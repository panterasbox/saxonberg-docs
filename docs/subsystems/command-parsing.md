# Command Parsing

The path between "the player typed text and pressed enter" and a
structured `ParseResult` the dispatch layer can match against. This
doc covers the tokenizer (`CommandLineApi`), the pluggable parser
shape (`Parser` / `ParseResult`), and the default `msh` shell.

For everything downstream of parse — verb matching, model binding,
controller execution — see [command-routing.md](./command-routing.md).

The shape lives in:

- `packages/server/src/mud/api/command-line.ts` — `CommandLineApi`,
  `RawToken`, `ParsedCommand`, `ParsedPipeline`, `format()`.
- `packages/server/src/mud/lib/command/parsers/<name>.ts` — pluggable
  parser modules. Today: `msh.ts` only.
- Parser-related types on `packages/server/src/mud/api/command.ts` —
  `Parser`, `ParserContext`, `ParseResult`.

## Pluggable parsers

Parsing is a per-actor choice. The actor's `shell.parser` setting
(declared on `CommandGiverMixin`, type `enum`, default `'msh'`)
names the parser; `CommandApi.resolveParser(spec)` dynamic-imports
the module:

- Bare name (no `/`) → `<src>/mud/lib/command/parsers/<name>.ts`.
- Absolute path (`/X`) → `<src>/mud/X.ts`.

The setting is enum-typed so unknown parser names are rejected at
`set` time. Adding a new parser is a one-line append to `enumValues`
on `CommandGiverMixin.settings`.

```ts
interface Parser {
  name: string;
  parse(text: string, ctx: ParserContext):
    ParseResult | Promise<ParseResult>;
}

interface ParserContext {
  commandGiver: Stuff & CommandGiver;
  location: Location;
  available: CommandDefinition[];   // recency-stack-filtered
}

interface ParseResult {
  parsed?: ParsedCommand;             // dispatcher runs match + assemble
  bound?: { command, model };         // parser already chose; skip match
  error?: string;                     // surfaces via auto-emit
}
```

The dispatcher dispatches on the first non-undefined field of
`ParseResult`. Token-based parsers like `msh` return `{ parsed }`;
intent-based parsers (LLM, NL) can short-circuit by returning
`{ bound: { command, model } }` — they pick the verb and produce
field values themselves, and the dispatcher skips parse + match,
running only resolve + execute. The `available` field on
`ParserContext` hands the parser the recency-stack-filtered command
set so it can constrain its choices to verbs the actor can actually
fire.

### Why pluggable

Hard-coding parse semantics into `executeCommand` would force every
shell to fit the same mold. The `Parser` interface decouples ingress:
token-based parsers return `{ parsed }`, intent-based parsers return
`{ bound }`. Future LLM- or NL-driven parsers slot in without
touching the dispatcher.

## The `msh` shell

`msh` (Mud SHell) is the default. It wraps `CommandLineApi.parsePipeline`
and emits `{ parsed: ParsedCommand }`. Pure tokenization — no game
logic, no Stuff awareness, no YAML knowledge.

### Lexical rules

**Whitespace.** Token boundaries are space, tab, newline, carriage
return; runs collapse. Empty input yields an empty pipeline (the
ingress layer short-circuits before reaching the parser).

**Double quotes.** `"…"` quotes a token; whitespace and reserved
characters inside are part of the token. Recognized escapes inside
`"…"`:

| Escape | Result |
|---|---|
| `\"` | `"` |
| `\\` | `\` |
| `\n` | newline |
| `\t` | tab |
| `\r` | carriage return |
| `\X` (other) | kept verbatim as `\X` |

**Single quote is literal.** `'` is a literal character, not a
quoting mark. This is a deliberate departure from POSIX shells: a
chat-driven MUD has apostrophes everywhere (`don't`, `'allo`, `lyon's
mane`). UI-side emphasis is markdown, not single-quoting. The `say`
verb's alias `'` works because `'hello world` lexes as `verb='`,
`args=['hello', 'world']`.

`\'` is not a recognized escape — `'` is just a literal character,
escaping is unnecessary.

**Adjacent-quoted concatenation.** `--name="hello "world` is one
token `--name=hello world` (bash convention; needed so quoted
segments can adjoin literal characters).

**Outside quotes:** `\<space>` (or any whitespace char) is literal
whitespace inside the token (single token, doesn't break on the
escaped space). `\\` is `\`. `\"` is `"` (lets you embed a `"` in
an unquoted token). Other `\X` keeps `\X` verbatim.

**Reserved characters.** Only `|` is reserved at the lex layer (the
pipeline separator). `;`, `&&`, `||`, `(`, `)`, backtick, `$` are
literal characters in tokens.

### Pipe boundary

`|` is a pipeline separator. The tokenizer splits at unquoted `|`
into multiple `ParsedCommand`s. Inside `"…"` and after a backslash-
escape the `|` is literal.

**Pipes are NYI.** If the resulting pipeline has more than one
`ParsedCommand`, `parsePipeline()` throws
`Error('Command piping is not yet implemented')` immediately. No
silent tolerance, no later-stage detection. The throw bubbles to
`executeCommand`'s catch arm and surfaces via the auto-emit's
warning path.

### `RawToken` classification

Tokens emerging from the tokenizer are classified, not free strings —
the matcher consumes the classification, not raw text:

```ts
type RawToken =
  | { kind: 'word', value: string, raw: string, pos: number }
  | { kind: 'short-flags', flags: string, raw: string, pos: number }   // "-vn5" → "vn5"
  | { kind: 'long-flag', name: string, raw: string, pos: number }      // "--verbose"
  | { kind: 'long-with-value', name, value, raw, pos }                 // "--name=Aslan"
  | { kind: 'stop-options', raw: string, pos: number };                // "--"
```

`pos` is the byte offset of the token's start in the original input
— used by greedy positionals to recover the "everything from here
on" slice without losing whitespace runs.

Classification rules:

- `--name=value` always splits on the first `=` → `long-with-value`.
- `--name` → `long-flag`.
- `^-[A-Za-z][A-Za-z0-9]*$` → `short-flags`. Everything else
  starting with `-` is a `word`. This is what makes `add -5` parse
  the `-5` as a positional rather than a flag stack.
- Bare `--` → `stop-options`.
- After `stop-options`, every subsequent token is forced to `word`
  regardless of leading characters.

The first token must be a `word` and is the verb. Verbs cannot
start with `-` or `--` — that's a parse error or "unknown verb"
depending on how the dispatcher labels it.

### Short-flag stacks

Short-flag tokens can collapse multiple flags into one cluster.
`-vn5` where `v` is boolean and `n` takes a value → `v: true, n:
'5'`. The matcher peels left to right; when it hits a value-bearing
flag, the rest of the token is the value, and any remaining
characters are not flags.

The cluster is a single `RawToken` of kind `short-flags` with
`flags: 'vn5'`; the matcher applies the per-flag YAML definitions
when binding.

## `format()` — round-trip

`CommandLineApi.format(parsed)` is the inverse of parse. Used for
audit / history rendering and for round-trip property tests:

```
parse(format(parse(t)))  ≡  parse(t)   for any valid t
format(parse(t))                     ─ canonical-form text
```

Quoting rules in `format`:

- Words containing whitespace, `"`, `\`, or unquoted `|` are emitted
  as `"…"` with `"` and `\` escaped.
- Apostrophes stay bare.
- Greedy field text round-trips by virtue of being captured as the
  raw input slice during binding (a matcher concern, not a parser
  concern).

`format` is not exported through `@saxonberg/types` — the client
input bar is plain text, widgets send structured payloads, and
neither needs the inverse. Server-side uses include audit/history
rendering and property tests.

## ParsedCommand shape

```ts
interface ParsedCommand {
  verb: string;
  rawTokens: RawToken[];   // verb at index 0 as kind:'word'
  source: string;          // slice of original input this command came from
  start: number;           // byte offset in original input
}

interface ParsedPipeline {
  commands: ParsedCommand[];
  raw: string;
}
```

`source` and `start` carry forward the byte-offset context the
greedy positional binder needs to slice the original input; without
them, greedy fields would lose interior whitespace runs.

## Cross-references

- [command-routing.md](./command-routing.md) — what happens after
  parse: verb matching against the recency stack, the chain-of-
  responsibility dispatch, MVC, schema delivery.
- [shell-environment.md](./shell-environment.md) — the `shell.parser`
  setting and how `EnvironmentMixin` + `resolveSetting` route per-
  actor settings.
