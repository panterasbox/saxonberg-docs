# Glob subsystem — fungible stacks

The glob subsystem models "a quantity of indistinguishable units" as
a single Stuff with an integer `quantity` field. A 30-coin stack is
one Stuff, not 30 sibling Stuffs; the framework treats it as 30
units at the contract surface (`drop 5 coins`, `30 coins are
here`) while persisting one row.

Pieces:

- **`GlobbableMixin`** (`lib/stuff/Globbable.ts`) — the substrate
  declaration. Adds `quantity: number`, the inter-Stuff contract
  methods (`getQuantity` / `setQuantity` / `canMergeWith` /
  `canSplit` / `onSplit` / `onMerged`), and the
  `fieldMeta`'s globIdentity entries static.
- **`Mixins.Globbable` / `MixinApi.isGlobbable`** — registry constant
  and predicate, same pattern as every other mixin.
- **`GlobbableApi`** (`api/glob.ts`) — `split`, `merge`, `canMerge`,
  and the `applyQuantity` workhorse used by every quantity-bearing
  controller.
- **`Stuff.getPresentation()`** — count-aware display naming
  (`"30 coins"`); pluralization defers to `GrammarApi.pluralize`.
  The count is an affix on the universal self-presentation render,
  not a Globbable-owned method, because presentation is every
  Stuff's concern.
- **`ContainmentApi.placeDirect`** — fresh-placement primitive used
  by split. Bypasses arrival/leave witnesses, capacity validators,
  and the merge-on-arrival ripple. Gated `ApiOnly`.
- **Merge-on-arrival ripple** — `ContainmentApi.move` fires a
  late-bound hook after `onContainableAdded`; the hook scans the
  destination for a mergeable sibling and absorbs the arrival.
- **MQL touchpoints** — `MqlOneResult` / `MqlManyResult` carry an
  optional `quantity: MqlQuantity`; formal `:{N}` / `:{*}` parses to
  a `QuantityNode`; natural-language `5 coins` / `all coins` rides
  on desugar's side-channel hint.

This file is the operational reference. The continuous-matter sibling
shipped as its own substrate — see [bulk.md](./bulk.md); the
structured-notes substrate (`quantity-clamped`,
`quantity-clamped-rejected`, `empty-result`, `target-declined`) is
defined in [response-envelope.md](./response-envelope.md).

---

## The mixin

```ts
export interface Globbable {
  getQuantity(): number;
  setQuantity(n: number): void;          // validates n >= 1, integer
  canMergeWith(other: Stuff): boolean;   // veto seam
  canSplit(n: number): boolean;          // veto seam
  onSplit(splitoff: Stuff): void;        // witness on source
  onMerged(absorbed: Stuff): void;       // witness on survivor
}
```

Persistent field: `quantity: number` (default 1). Cloning a template
with `data.quantity: 50` produces a 50-stack; omitting it defaults
to 1.

`onSplit` and `onMerged` ship as no-op terminals so subclasses can
`super.onSplit(splitoff)` without ceremony — same shape as
`Stuff.onDestruct()`.

### `globIdentityFields ⊂ persistentFields`

A class declares a (possibly empty) subset of its `fieldMeta`'s persistent entries
that defines glob identity:

```ts
class Coin extends GlobbableMixin(Thing) {
  static fieldMeta: FieldMeta = {
    tarnished: { persistent: true },
    denomination: { persistent: true },
    lastTouchedAt: { persistent: true },
  };
  static fieldMeta: FieldMeta = {
    tarnished: { globIdentity: true },
    denomination: { globIdentity: true },
  };
  // ...
}
```

Two stacks of `Coin` merge iff:

1. Same template path (`stuff.getTemplatePath()`).
2. Neither side has shadows.
3. Neither side has attached adornments.
4. Equal values for every field in the **union** of both classes'
   `fieldMeta`'s globIdentity entries.

Subclasses extend the parent's list:
a `{ globIdentity: true }` entry per field; a subclass adds its own
rather than spreading the parent's, because `getAllFieldMeta` already
unions up the chain.

The framework verifies `globIdentityFields ⊂ persistentFields` at
class-registration time via `MixinApi.assertComposable` —
runtime-only fields can't define glob identity (they wouldn't survive
save/load and the two "matching" stacks would diverge after a
reload).

### Composition constraints

Three constraints enforced at first registration via
`__validateComposition__` (called by `MixinApi.assertComposable`):

- `Globbable ⊥ Container` — a glob is not a container.
- `Globbable ⊥ Singleton` — singletons are one-instance-per-
  templatePath; `split` produces a sibling at the same path that
  `StuffApi.clone` would refuse for a singleton class.
- `globIdentityFields ⊂ persistentFields` — identity fields must
  round-trip through hydration.

All three throw at first instance registration; the check is
memoized per concrete class identity (HMR-aware: replacing a class
binding re-validates against the new chain).

### Defaults

- `globIdentityFields = []` is legal and means *every* instance of
  this glob is mergeable with every other (strict template-fungibility).
- Coin-with-fields above behaves more carefully — only same-denomination
  same-tarnish stacks merge.

> ⚠⚠ **The shipped `obj/Coin` keys on `['currency', 'denomination']`, and
> the currency half is load-bearing.** Denomination identity is
> `(currency, faceValue)`, so without the currency in the key two issuers'
> like-valued coins would **merge into one stack — creating money by a
> merge**, with no ledger row and no error. Glob identity is the defence
> here, not a rule someone remembers. Any future value-bearing glob (scrip,
> a bearer token) inherits the same obligation. See
> [banking.md](./banking.md).

---

## `GlobbableApi.split`

```ts
GlobbableApi.split(source, n): Promise<Stuff & Globbable>
```

- Validates `n` is a positive integer ≤ `source.getQuantity()`.
- Calls `source.canSplit(n)` (shadow seam); throws on veto.
- **Whole-stack short circuit**: `n === source.getQuantity()` returns
  `source` itself with no clone (the caller is going to move the
  whole stack).
- Otherwise:
  - `StuffApi.clone(source.getTemplatePath())` → splitoff.
  - Copies every value in `source.constructor.globIdentityFields`
    onto the splitoff via the public method surface (`getX` / `setX`
    if present, falling back to direct property access).
  - Sets `splitoff.setQuantity(n)`; `source.setQuantity(M - n)`.
  - The per-field copy goes through `StuffApi.copyField` — a general
    Stuff-to-Stuff inter-stuff-contract primitive (`getX` / `setX`
    when present, property access fallback).
  - **Places the splitoff via `ContainmentApi.placeDirect`** into
    `source.getContainer()` (codebase method name — the slate calls
    it `getEnvironment` colloquially). No arrival witnesses fire.
  - Fires `source.onSplit(splitoff)`.

Decorated `@CallSecurity(SecurityPolicies.ApiOnly)` because the
placeDirect bypass is too powerful for non-Api callers.

The split is **within-container subdivision, not movement**. The
matter was already in the room before the split; subdividing it
doesn't constitute "arrival." That's what `placeDirect` ensures.

---

## `GlobbableApi.merge`

```ts
GlobbableApi.merge(survivor, absorbed): void
```

- Validates both are Globbable and that `survivor.canMergeWith(absorbed)`
  returns true.
- `survivor.setQuantity(survivor.getQuantity() + absorbed.getQuantity())`.
- `StuffApi.destruct(absorbed)` — fires its `onDestruct` chain,
  including the Containable cleanup that unhooks absorbed from its
  container. Subscribers that want "this Stuff went away" hook here.
- Fires `survivor.onMerged(absorbed)`.

`merge` itself emits no movement events. Two integration sites:

1. **Merge-on-arrival ripple** in `ContainmentApi.move`. After the
   post-move `on*` witnesses, the late-bound hook scans the
   destination for a mergeable sibling and calls `merge`. Ordering:
   merge **after** `onContainableAdded` so subscribers see the
   arrival as a distinct event before the destruct.
2. **Reglob** inside `applyQuantity`. When the action callback
   returns `{ ok: false }` for a candidate that was split, the
   operand is folded back into the source.

Decorated `@CallSecurity(SecurityPolicies.ApiOnly)`.

---

## `ContainmentApi.placeDirect`

```ts
ContainmentApi.placeDirect(item, env): void
```

Sibling to `move` for the "matter was already there" case:

| | `move` | `placeDirect` |
|---|---|---|
| Departure events (`onContainableRemoved`) | Fires | None |
| Arrival events (`onContainableAdded`) | Fires | None |
| `onMoved` on item | Fires | None |
| Capacity validators | Run | Bypassed |
| Mixin compatibility (Containable / Container) | Validated | Validated |
| Merge-on-arrival ripple | Triggers | Bypassed |
| Accepts existing-container item | Yes | **No** — throws |
| Security | (none) | `ApiOnly` |

**Fresh-placement precondition**: `item.getContainer() === null`.
Existing-container Stuffs must go through `move`. This single check
rules out smuggling (moving an item past leave-witnesses) and
teleport-past-guard (relocating without arrival-witnesses).

Use cases:

- Glob split (splitoff is freshly cloned).
- First-placement bootstrap paths after `StuffApi.clone`.
- Hot-reload reattach (post-clone, pre-relink).

---

## `GlobbableApi.applyQuantity`

The workhorse every quantity-bearing controller routes through.

```ts
applyQuantity<R>(
  candidates: Stuff[],
  quantity: MqlQuantity,
  action: (operand: Stuff, applied: number) => Promise<
    | { ok: true; payload: R }
    | { ok: false; reason: string }
  >,
  opts: { field: string; query?: string }
): Promise<{
  ok: boolean;
  applied: number;
  status?: 'partial' | 'declined';
  notes: Note[];        // canonical @saxonberg/types shape
  payloads: R[];
}>
```

`opts.field` is **required** — it stamps the originating field name
onto every emitted note so controllers can forward `result.notes`
straight into `ctx.note(...)` without re-shaping. `opts.query`
carries the raw user-typed query string for `empty-result` /
`match-ambiguous` notes.

Behavior:

- **Empty candidate list** → immediate `{ ok: false, status:
  'declined' }` with an `empty-result` note. No actions run.
- **Strict pre-check** (`mode: 'strict'`, `kind: 'count'`):
  `sum(units across candidates) < n` → immediate decline with
  `quantity-clamped-rejected`. No actions run. Non-globbable
  candidates contribute 1 unit each; globbable candidates contribute
  up to their full quantity.
- **All-kind**: action runs on every candidate at full contribution.
- **Count-kind**: walk in scored order; for each candidate
  `contribution = min(units(c), remaining)`. Split when globbable
  and `contribution < c.getQuantity()`, else operand = c.
- **Action `ok: false`**: emit a `target-declined` note (target =
  the candidate, *not* the post-split operand). If a split occurred,
  reglob (merge operand back into c). Continue the walk; remaining
  is unchanged.
- **Lenient overflow** (`mode: 'lenient'`, count kind, remaining > 0
  after walk): emit `quantity-clamped`; status `'partial'`.
- **Status rule**: `'partial'` when any progress was made and
  something diverged (target-declined or lenient clamp);
  `'declined'` when `applied === 0`; absent on a clean run.

Throw propagation (G5): if `action` throws, the helper propagates.
Throws are programmatic violations; `{ ok: false }` is the
soft-failure signal.

### Note shape

`applyQuantity` emits four canonical `Note` kinds from
`@saxonberg/types`, pre-stamped with `field` from `opts.field`:

- `quantity-clamped { field, requested, applied }`
- `quantity-clamped-rejected { field, requested, available }`
- `empty-result { field, query }`
- `target-declined { target: StuffRef, reason }`

`target-declined.target` is a `StuffRef` (constructed via
`MessageApi.refOf`) — never a raw `Stuff` object — so the note is
wire-safe out of the box. Controllers forward `result.notes`
straight into `ctx.note(...)`; no re-shaping. Full shapes and
auto-escalation rules live in
[response-envelope.md § Notes](./response-envelope.md).

### Mode is transport-only

`MqlQuantity.mode` carries the syntax-form signal (formal `:{N}` →
strict; natural-language `2 X` → lenient). **Controllers don't read
`mode`** — they pass the whole `MqlQuantity` through to
`applyQuantity` without branching. The helper consumes it (strict
pre-check vs lenient clamp). The one exception is a controller
deliberately overriding policy
(`applyQuantity(stuff, { ...quantity, mode: 'strict' }, action)`),
a loud explicit signal; v1 has no such verbs.

---

## Display rendering — count folds into `Stuff.getPresentation()`

Count-aware naming is an affix on the universal self-presentation
render — not a `GlobbableApi` concern and not a separate method:

```ts
stuff.getPresentation(): string
```

Returns `count + " " + plural` when `stuff` is globbable with
`getQuantity() !== 1`; otherwise the bare name/shortDescription
chain. Pluralization defers to `GrammarApi.pluralize`, which
respects host-side `getPluralForm()` overrides for irregulars
(`"mouse"` → `"mice"`).

```
1-stack coin   → "coin"
30-stack coin  → "30 coins"
3-stack mouse  → "3 mice"        (Mouse declares getPluralForm)
non-globbable rock → "rock"
```

The recognition pipeline (recognition slate) composes on top of
this baseline — weaving the count into the identity layer with
viewer-side perception and recognition state.

---

## MQL touchpoints

Two parallel input paths, one resolved value:

| Form | Audience | Mode | Semantics |
|---|---|---|---|
| `drop 2 roses` (natural language) | Humans | `lenient` | Desugar side-channel hint. Clamp on shortfall. |
| `drop roses:{2}` (formal) | Composers, scripts | `strict` | Parser `QuantityNode`. Pre-check; decline on shortfall. |
| `drop all roses` | Humans | `lenient` | Same as `2 roses` but `kind: 'all'`. |
| `drop roses:{*}` | Composers | `strict` | Same as `:{N}` but `kind: 'all'`. |

Both land on the same `MqlQuantity` slot:

```ts
type MqlQuantity = {
  value: { kind: 'count'; n: number } | { kind: 'all' };
  mode: 'strict' | 'lenient';
};

interface MqlOne {
  stuff: Stuff | null;
  via?: MqlMatchVia;
  quantity?: MqlQuantity;       // ← here
}
```

See [mql-grammar.md](../mql-grammar.md) and [mql.md](./mql.md) for the
grammar / pipeline mechanics.

### Singular field + quantity hint (v1 note)

Controllers that opt in to quantity should declare `type: objects`
on the YAML field. A singular `type: object` field with a quantity
hint will see `MqlOneResult.quantity` set, but the dispatcher
doesn't synthesize a list of one — the controller would have to wrap
it. v1 verbs that take quantity (`drop`, `get`) declare `type:
objects`.

---

## Author guide

To compose Globbable on a host:

```ts
class Coin extends GlobbableMixin(ContainableMixin(NamedMixin(Idea))) {
  static _mixinName = 'Coin';
  static fieldMeta: FieldMeta = {
    quantity: { persistent: true },
    name: { persistent: true },
    denomination: { persistent: true },
    tarnished: { persistent: true },
  };
  static fieldMeta: FieldMeta = {
    denomination: { globIdentity: true },
    tarnished: { globIdentity: true },
  };

  public denomination: 'gold' | 'silver' | 'copper' = 'copper';
  public tarnished: boolean = false;
}
```

Then in the template:

```yaml
# /obj/item/Coin/seed.yaml
path:  /obj/item/Coin
class: /lib/glob/Coin   # illustrative — Coin is a test fixture; the
                        # shipped substrate is the GlobbableMixin in
                        # lib/stuff/Globbable.ts (no lib/glob/ dir)
data:
  keywords: [coin, gold]
  quantity: 1      # default; clone-time override is common
```

Clone-time `data.quantity: 50` produces a 50-stack.

### Verb-side wiring

Controllers that want to participate in quantity-bearing input
follow the two-phase shape:

```ts
async execute(model, context): Promise<void> {
  const { stuff, quantity, raw } = model.targets;
  if (!quantity) {
    await this.executeWholeSet(stuff, raw, context);
    return;
  }
  const result = await GlobbableApi.applyQuantity(
    candidates,
    quantity,
    async (operand, applied) => {
      ContainmentApi.move(operand, context.location);
      return { ok: true, payload: { operand, applied } };
    },
    { field: 'targets', query: raw }
  );
  for (const note of result.notes) context.note(note);
  this.renderProse(result, raw, context);
}
```

`DropController` and `GetController` are the canonical examples;
mirror their structure.

In v1, the action callback always returns `ok: true` for `drop` /
`get` because `ContainmentApi.move` throws on programmatic-contract
failure (no soft failure to signal). Capacity-driven `ok: false`
arrives with the collision slate.

---

## Limitations / open questions

Trade-offs and deferred work documented for future maintainers:

- **Merge-on-arrival is O(siblings).** Every `ContainmentApi.move`
  into a container holding any Globbable contents walks the contents
  list looking for a mergeable sibling. Fine at v1 scale; if a
  container ends up with many distinct glob kinds (a treasury), a
  per-container index keyed on `templatePath` takes the scan to
  O(1) — cheap to maintain on add/remove. Land that index when
  measurement shows the scan biting.
- **Initial-state merge sweep is deferred.** When bootstrap loads a
  world whose YAML seeds two mergeable stacks into the same
  container, they aren't merged on load — the merge-on-arrival
  ripple fires on `move`, and the seeded objects don't move. The
  honest fix is a `PostRegistration` sweep keyed by container. Until
  then the contract is "don't seed two mergeable stacks in the same
  container"; content-authoring tools should warn (separate work).
- **Sibling visibility during the split-active transient.** Between
  `GlobbableApi.split` and the action's downstream `move`, the
  source's environment momentarily contains two mergeable siblings.
  Event-driven subscribers don't observe this — split bypasses
  arrival witnesses by design. *Static* observers (anyone querying
  the env's contents collection directly during the window) see
  both stacks; counts add up correctly, so the existing-state view
  is consistent. Subscribers that want to react specifically to
  splits hook `onSplit` on the source instead of trying to observe
  the transient.
- **MQL `coin:[5]` indexes the match set, not units within a stack.**
  Stacks are single Stuffs, so `coin:[5]` against a 30-stack
  produces nothing (there's only one match in the set). Players
  don't have an ordinal intuition for "the 5th coin in a pile";
  `5 coins` / `coin:{5}` are the supported shapes for "five units
  from the pile." Documented in [mql-grammar.md](../mql-grammar.md).

## Future extensions

- **Bulk form** — continuous matter (water, coffee) **shipped** as a
  separate substrate, NOT as a Quantity-valued glob. The earlier
  fluid-as-Stuff sketch (a `Quantity<U>` count split via `placeDirect`,
  merged on arrival — the structural mirror of Globbable) was retired:
  bulk is **not a Stuff** at all. It is a `{ material, amount }`
  attribute of its holder, moved by `BulkableApi.transfer`. Glob and
  bulk share only the `MqlQuantity` discriminated union (glob uses
  `count` / `all`; bulk uses the `measure` variant) and the reused
  response-note kinds. See [bulk.md](./bulk.md).
- **The recognition pipeline** composes on top of the count affix in
  `Stuff.getPresentation()`. It weaves count, perception-filtered
  visibility, viewer-side recognition, and bucket-keyed verbosity in
  one viewer-aware step. Globbable contributes data (`getQuantity`,
  the host's presentation and optional `getPluralForm`); it doesn't
  know about viewer state. See
  recognition-slate.md.

## Antipatterns

- **Glob inside a glob.** Globs aren't `Container`; the composition
  validator catches it at registration. If you want a containment-
  bearing glob, you want a different abstraction (probably bulk —
  see [bulk.md](./bulk.md)).
- **Instance-unique state on a stack.** Shadows and adornments
  disqualify a stack from merging. If you find yourself wanting
  per-instance details on a glob (a unique scratch on one coin),
  the host class should not be Globbable — or the unique instance
  should be its own non-globbable Stuff.
- **Stuff-count capacity on glob-bearing containers.** A bag rated
  for "5 items" that holds a 30-coin pile shouldn't refuse to let
  you split the pile. Glob-bearing containers should declare
  capacity in units, mass, or volume — see the capacity model in
  the future collision slate.
- **Reaching past `placeDirect` for fresh placement.** Don't write
  `item.setContainer(env)` / `env.addContainable(item)` directly,
  even for fresh-Stuff cases. `placeDirect` packages the right
  invariants; the chokepoint is gated `FromContainmentApi`.
- **Branching on `MqlQuantity.mode` in a controller.** Mode is
  transport-only; pass it through to `applyQuantity`. The strict
  vs lenient policy difference is in the helper, not the verb.

---

## Cross-references

- [mql.md](./mql.md) — pipeline (`QuantityNode`, desugar signature,
  resolver passthrough).
- [../mql-grammar.md](../mql-grammar.md) — user-facing grammar:
  natural-language quantity prefix, formal `:{N}` / `:{*}`.
- [collections.md](./collections.md) — collection-shape canon for
  related mixins (Set / Map / list).
- [bulk.md](./bulk.md) — the continuous-matter sibling
  (holds-as-attribute, not a Stuff); shares the `MqlQuantity` union and
  the structured-notes kinds.
- [response-envelope.md](./response-envelope.md) — the
  structured-notes substrate `applyQuantity` emits into; canonical
  shapes for the four glob note kinds.
- ../slates/tails/recognition-slate.md —
  where the recognition pipeline composes count + perception +
  recognition on top of `getPresentation()`'s count affix.
