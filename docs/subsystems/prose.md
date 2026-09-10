# Prose Templating

`ProseApi` (`mud/api/prose.ts`) is the rendering layer for any prose
that lives outside source — schema-declared settings, CMS-authored
room/NPC/item descriptions, future prompts, future combat narration.
It turns a string template with `{{ var }}` placeholders, `{% if %}`
conditionals, and `| filter` chains into a finished `Mml` fragment.

Built on [LiquidJS](https://liquidjs.com/), the same Liquid dialect
used by Shopify, Jekyll, and GitHub Pages — well-established syntax,
sandboxed by construction, no `eval`, no filesystem access by default.

## Where it sits

`Mml` and `ProseApi` are orthogonal concerns:

- **`Mml`** answers "what semantic markup wraps this fragment?" —
  `<name>`, `<location>`, `<direction>`, eventual CSS/layout.
- **`ProseApi`** answers "given this template and these vars, what
  string should I produce, and which conditional segments survive?"

A prose-producing path uses both: `ProseApi` decides what fragments
to splice in, `Mml` provides the markup wrappers.

| Use ProseApi when | Use `Mml.compose` when |
|---|---|
| Players might want to override the verbiage | The output is dev-owned and structural |
| Authors / scripts override per-room or per-object | The block has multi-line layout |
| Prose has multiple audience variants (self / target / witness) | Inline tagged-template prose is fine |
| Stable one-liner with interpolation slots | E.g. `LookController`, `InventoryController` |

## Mml-aware output

The Liquid engine is configured with a custom output handler that
mirrors `Mml.compose`'s interpolation rules:

- `Mml` fragments emit verbatim (already escaped).
- Strings, numbers, booleans are five-entity escaped via `Mml.escape`.
- `null` / `undefined` (including missing variables) → empty string.
- Objects with `toMml()` are unwrapped; non-`Mml` returns are escaped.

So you can pass a pre-rendered `Mml.actor(actor)` as a variable and it
threads through the template without double-escaping. The render
output is wrapped via `Mml.fromMarkup` so callers receive a real
`Mml` value.

## Surface

```typescript
class Prose {
  static parse(source: string): Prose;          // compile once
  render(vars: Record<string, unknown>): Mml;   // execute many times
  toString(): string;                           // raw source
}

class ProseApi {
  static format(source: string, vars: Record<string, unknown>): Mml;
  static registerFilter(name: string, fn: FilterFn): void;
}
```

`Prose.parse` for hot paths where the template is constant (the
movement-message defaults, prompts) — compiling once and rendering
many times skips re-parsing on each call. `ProseApi.format` is the
one-shot convenience for ad-hoc renders.

## Engine configuration

```typescript
new Liquid({
  strictVariables: false,        // missing var → ''
  strictFilters:   true,         // unknown filter → throws
  ownPropertyOnly: true,         // never resolve through prototype chain
  outputEscape:    outputEscapeMmlAware,
});
```

`strictVariables: false` matches the previous `Mml.format` semantics —
typos in variable names render empty rather than blowing up at render
time. `strictFilters: true` does the opposite for filters: an unknown
filter is almost always a typo, and silently dropping a filter chain
produces hard-to-debug output.

`ownPropertyOnly: true` is a sandboxing choice — variable resolution
never walks the prototype chain, so a passed-in object can't expose
unintended methods or fields.

File-loading tags (`{% include %}`, `{% render %}`, `{% layout %}`)
work only with a filesystem root, which we never configure — referring
to them throws.

## Default filters

Registered at module init in `prose.ts`. All take an input from the
left side of the pipe and may take additional positional arguments.

Filter input expectations split between *strict* (mixin required;
filter renders empty otherwise) and *lenient* (any Stuff accepted;
the underlying API supplies a last-stitch fallback):

| Filter | Input contract |
|---|---|
| `name` | strict — `Named` only; non-Named renders empty |
| `pronoun`, `possessive` | strict — `Gendered` only |
| `item`, `location`, `object`, `article` | lenient — any Stuff |
| `direction` | non-empty string |
| `cap` | any non-null value (coerced to string) |
| `quantity` | strict — value with a `toMml()` method (`Quantity`); emits `<quantity>` markup, else empty |
| `quantity_canonical` | strict — value with a `formatMml()` method (`Quantity`); emits canonical `<quantity>` markup, else empty |

`name` is strict because there's no honest fallback for "this thing's
proper name" — wrapping an `'a sword'`-style fallback in `<name>`
tags would lie about identity. The other Mml-vocabulary filters
delegate to `Stuff.getPresentation()` via `Mml.thing` / `.location`
/ `.object`, which already drop to sensible defaults (`'an item'`,
`'somewhere'`, `'something'`) when the Stuff has neither `Named` nor
`Visible`; the filter lets that fallback surface rather than
swallowing the call.

### Mml vocabulary

`{{ stuff | name }}` — Named only; renders an empty placeholder
otherwise.

`{{ stuff | item }}`, `{{ stuff | location }}`, `{{ stuff | object }}`
— wrap any Stuff's display name in the corresponding markup tag with
`stuff-id` attribution. Equivalent to calling `Mml.thing(stuff)` etc.
directly; uses the Mml factory's last-stitch fallback when no display
surface is composed.

`{{ direction | direction }}` — wraps a raw direction string in
`<direction>` markup. Note the filter shadows the variable name; both
are spelled the same.

### Grammar (`GrammarApi`)

`{{ text | cap }}` — capitalize the first character. Operates on raw
strings only; passing an `Mml` fragment will cap the leading `<` and
mangle the markup.

`{{ stuff | pronoun }}` / `{{ stuff | pronoun: 'obj' }}` — return a
pronoun for a `Gendered` stuff. Default kind is `'subj'`; other kinds
are `'obj'`, `'poss'`, `'reflex'`. The pronoun set is selected from
the `Pronouns` enum (`He` / `She` / `They` / `It`) on
`GenderedMixin.pronouns`. Non-Gendered stuff renders empty.
`GrammarApi.pronoun()` (called directly from TS) falls back to neuter
on non-Gendered input rather than empty.

`{{ stuff | possessive }}` — alias for `pronoun: 'poss'`.

`{{ stuff | article }}` — return `'a'` or `'an'` for any Stuff based
on a vowel-onset heuristic against the display name; falls back to
`'a'` when no display surface is composed. Phonetic exceptions
(`an honest`, `a unicorn`) need per-stuff overrides; not handled in
v1.

## Authoring patterns

**Conditional segment** — render a piece only when a var is present:

```
You arrive{% if direction %} from the {{ direction }}{% endif %}.
```

**Conditional with else**:

```
{% if combat %}[fighting]{% else %}[idle]{% endif %}
```

**Filter chain** — mix grammar and Mml vocabulary:

```
{{ actor | pronoun: 'subj' | cap }} attacks {{ target | name }}.
```

**Per-audience movement message** (canonical example, in
`MobileMixin`):

```
You arrive{% if direction %} from the {{ direction }}{% endif %}.
{{ mover }} arrives{% if direction %} from the {{ direction }}{% endif %}.
```

The directional/bland split that used to require two settings keys
collapses into a single template using `{% if direction %}`.

## Registering custom filters

```typescript
ProseApi.registerFilter('shout', (v) => String(v).toUpperCase());
```

Filter functions take `(input, ...args)`. Any return value is run
through the Mml-aware output handler — return an `Mml` fragment to
emit markup verbatim, return a raw string to get five-entity escaping.

## What we're NOT using (yet)

LiquidJS supports much more than we currently exercise:

- **Loops** (`{% for x in list %}`) — useful for prompt status
  effects, party listing, inventory rendering. Available, just unused
  today.
- **Math filters** (`divided_by`, `times`, `round`, etc.) — relevant
  if/when prompts grow numeric thresholds.
- **`{% capture %}`** — assign a rendered block to a local. Available
  but no use case yet.

These are all enabled; they just don't appear in current templates.

## Cross-references

- [messaging.md](./messaging.md) — `Mml`, Scene composer, where
  `ProseApi`-rendered fragments end up
- [shell-environment.md](./shell-environment.md) — settings keyspace
  that hosts most player-overridable templates today

## Where the engine lives

The Liquid engine and the default filter set live in `ProseLogic`
(`platform/idea/api/ProseLogic.ts`), not in `lib/prose/Prose.ts`. `liquidjs` is a
dependency outside `src/mud/`, and under [the import
boundary](../architecture.md) only the Api tier may import it.

`Prose` keeps the value-object half — the source string, an opaque
`CompiledProse` handle, and the `Mml`-typed `render` surface callers
speak — and reaches back through `ProseApi.compile` /
`renderCompiled` / `registerFilter`. Author-facing behaviour is
unchanged: `ProseApi.format` and `ProseApi.registerFilter` are still the
surface, and the Mml-aware output escaping and filter semantics moved
verbatim.

One consequence worth knowing: the memoised engine is now module state
in a **hot-reloadable** module, so `dest /platform/idea/api/prose` rebuilds it and
drops any filter installed at runtime through `ProseApi.registerFilter`
(the default set is rebuilt either way). Nothing in-tree registers one
today; an author who does should re-register after a reload. Making them
survive wants a durable filter registry — more machinery than the
current zero callers justify.
