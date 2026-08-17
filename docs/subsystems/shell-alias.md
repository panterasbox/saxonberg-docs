# Shell Aliases

`AliasMixin` is the per-instance command alias suite — verb-position
substitution with positional argument interpolation, plus a small
defaults tier walked from any composed mixin's static data. It's a
sibling of [shell-environment.md](./shell-environment.md) under the
`lib/shell/` substrate, and is composed onto `ShelledCharacter`. NPCs
that don't run an interactive shell don't carry it.

The shape lives in:

- `packages/server/src/mud/lib/shell/Alias.ts` — `AliasMixin`,
  `Alias` interface, `AliasEntry`, `AliasOverridesSnapshot`,
  `DefaultAliasEntry`. The collection-shape contract is **Shape B
  (keyed map)** per [collections.md](./collections.md).
- `packages/server/src/mud/api/shell.ts` — `ShellApi.expandAliases`,
  the cross-cutting helper invoked by the command pipeline. No new
  Api class; expansion sits next to `expandVariables` and
  `lookupSyntheticVar` because the shell suite shares one substrate
  Api.
- `packages/server/src/mud/cmd/shell/alias.yaml` — the `alias` player
  verb (subcommanded: `list / get / set / unset / describe`).
- `packages/server/src/mud/obj/command/shell/AliasController.ts` — the
  controller that operates on the host's `Alias` surface.

## Core Principle

Aliases are **verb-replacement only**. They expand exclusively at
the verb position of a `ParsedCommand` and substitute a single
command (no pipes, no scripting, no multi-step sequencing). The body
is text, re-tokenized at expansion time through the same
`CommandLineApi.parsePipeline` the rest of the shell uses — quoting,
escapes, and flag classification all behave identically inside the
body.

Defaults are declared on mixins via `static defaultAliases`,
exactly mirroring how `static settings`, `static syntheticVars`,
and `static commandContributions` work. There is **no central
registry**; the resolution walk picks contributions up by
prototype-chain inspection.

## Concepts

### Two-tier overrides + a defaults tier

Three layers, with most-specific winning:

| Tier | Storage | Lifetime |
|---|---|---|
| Defaults | `static defaultAliases` on a mixin layer | Compile-time |
| Persistent | `aliases` field on the host (Hydrator-saved) | Persisted |
| Session | `aliasesSession` on the host | In-memory only |
| Tombstones | a `null` value at a name in the persistent tier | Persisted |

Lookup order: session → persistent → tombstone-check → defaults.

### Tombstones

`unalias` (the `alias unset` subcommand, or programmatic
`removeAlias`) needs different semantics depending on what's at the
name:

- A session override → drop the override; persistent (or default)
  comes back.
- A persistent override → drop it. If the name ALSO has a default,
  write a tombstone (`null`) so the default doesn't silently
  resurrect on the next reboot. Otherwise just delete.
- No override but a default exists → write a tombstone outright.
- Nothing at all → returns false.

A tombstoned default reads as "not present" for every API surface
(`getAlias`, `getAliases`, `hasAlias`). Writing a new override at a
tombstoned name simply replaces the null with the new body.

### Bypass prefix

`\verb` runs the real verb even when an alias of the same name
exists. The leading `\` is stripped at `ShellApi.expandAliases`'s
front gate; alias lookup is skipped for that call only. Bash
convention; integrates with the existing `\X`-keeps-verbatim
tokenizer rules cleanly. The msh tokenizer collapses `\\` to a
single `\` at lex time, so `\\look` and `\look` produce the same
post-bypass verb (`look`); the spec-level "literal-backslash verb"
hypothetical is unreachable through msh and that's a known
limitation.

### Cycle guard + depth ceiling

Recursive expansion uses a per-call `Set<string>` of in-flight
alias names; re-entering the same name terminates that branch
silently (no throw — bash semantics, the inner reference would have
run the real verb). A hard depth ceiling of 16 hops is enforced as
a backstop and emits a soft `MudlogApi.warn` to the giver when
reached.

## Pipeline placement

```
[client] CommandBar onSend(text)
   │  ...
   ▼
avatar.executeCommand(text, opts)              ← CommandGiverMixin
   │  ...
   ├─ resolveActorParser(actor) → Parser
   ├─ parser.parse(text, parserCtx) → ParseResult
   │
   ├─ if ParseResult.parsed AND isAlias(giver):
   │     ShellApi.expandAliases(parsed, giver) → ParsedCommand
   │     (sets context.aliasExpansion when an alias fired)
   │
   ├─ if ParseResult.parsed: _runChain(parsed, ctx)
   ├─ if ParseResult.bound : skip parse + match ; aliases NOT applied
   │     (LLM/intent parsers chose the verb directly)
   ▼
   ...
```

Three guarantees the placement enforces:

1. **Token-level operation.** Expansion runs on classified
   `RawToken`s, not raw text. The tokenizer's existing quoting and
   escape rules apply once, at parse time; the expander reuses the
   classification rather than re-implementing it.
2. **Parser-agnostic.** The expansion logic doesn't care which
   parser produced the `ParsedCommand`. An LLM parser that emits
   `{ bound }` already chose the verb; aliases are out of scope on
   that path and the dispatcher correctly skips them.
3. **Mixin-gated.** NPCs without `AliasMixin` don't pay the cost.
   The `MixinApi.isAlias(giver)` predicate is the single guard.

## Algorithm

### Outer entry point

```
expandAliases(parsed, giver):
  if parsed.verb starts with '\\':
    return { parsed: stripBypass(parsed) }   // no expansion event
  inflight = new Set(); chain = []
  result = expandRecursive(parsed, giver, inflight, depth=0, chain)
  if chain is empty: return { parsed: result }   // no alias fired
  return { parsed: result, expansion: { aliasName, originalText, expandedText, chain? } }
```

### Recursion

Per-call `inflight: Set<string>` and `depth` counter. At depth 16 a
soft warn fires and recursion bottoms out at the last-stable
`ParsedCommand`.

```
expandRecursive(parsed, giver, inflight, depth, chain):
  if depth >= 16:
    warn (when giver is Sensor); return parsed
  entry = giver.getAlias(parsed.verb)
  if not entry: return parsed                         // termination
  if inflight.has(parsed.verb): return parsed         // cycle break

  inflight.add(parsed.verb); chain.push(parsed.verb)
  try:
    step = expandOnce(parsed, entry)
    next = buildResultParsed(step, parsed)
    return expandRecursive(next, giver, inflight, depth+1, chain)
  finally:
    inflight.delete(parsed.verb)
```

### One expansion step

```
bodyTokens = CommandLineApi.parsePipeline(entry.body).commands[0].rawTokens
userArgs   = parsed.rawTokens.slice(1)
consumed   = false

expandedBodyTokens = bodyTokens.flatMap(tok →
  match tok.kind:
    'word':
      if value matches /^\$\{?@\}?$/:
        consumed = true
        emit userArgs.slice()                         // multi-token
      else:
        { text, hadRef } = applyPosSubsToString(value, userArgs)
        if hadRef: consumed = true
        emit { kind:'word', value:text, raw:text, pos:0 }
    'long-with-value':
      { text, hadRef } = applyPosSubsToString(tok.value, userArgs)
      if hadRef: consumed = true
      emit { ...tok, value:text, raw:`--${name}=${text}`, pos:0 }
    other (short-flags / long-flag / stop-options):
      pass through unchanged
)

mergedTokens = consumed
  ? expandedBodyTokens                                // user-args consumed
  : [...expandedBodyTokens, ...userArgs]              // append (csh-style)
```

The "consume if referenced, append otherwise" rule is csh, not bash.
It's more intuitive for our use case: `alias greet = "wave at $1"`
behaves the way the player expects (`greet bob` → `wave at bob`),
and `alias l = look` simply prefixes (`l north` → `look north`).

### Positional substitution

| Form | Meaning |
|---|---|
| `$1` … `$9` | user-arg at that 1-based index, empty string if absent |
| `${1}` … `${9}` | brace form, same meaning |
| `$@` | all user-args, each preserved as its own token (multi-token) when naked; space-joined string when embedded |
| `${@}` | brace form, same meaning as `$@` |

Substitution applies to **`word.value`** and
**`long-with-value.value`** ONLY. Short-flag stacks (`-vn5`) and the
NAME of long-flags / long-with-values pass through untouched —
those positions don't carry user-meaningful text.

`$0`, `$#`, `$*` are intentionally not supported in v1; they're
better modeled as synthetic vars than as positional siblings if
demand surfaces.

### Source reconstruction (greedy fidelity)

The `source` field on `ParsedCommand` is what greedy positionals
read to recover whitespace runs. After expansion the original source
no longer aligns with the post-expansion tokens, so a synthetic
source is built:

- **Append case** (no positional refs): `body_text + (user's literal
  post-verb slice)`. Preserves user-typed whitespace verbatim — the
  user's typing for the trailing args is lossless.
- **Consume case** (positional refs fired): the merged token stream
  is canonicalized via `CommandLineApi.format`, which re-quotes
  values that need it. Quoted user-args keep interior whitespace
  through `format()`; unquoted multi-space between args is lost,
  but that whitespace was already collapsed by the tokenizer
  (before expansion ever ran), so the contract is no weaker than
  the no-alias baseline.

Both cases re-tokenize the synthesized source via
`parsePipeline` so the resulting `ParsedCommand` has clean `pos` /
`raw` fields on every token.

## Visibility — controllers and audit

When alias expansion fires, the dispatcher writes the following
record onto the live `CommandContext`:

```ts
interface AliasExpansionInfo {
  aliasName: string;          // the verb the user typed
  originalText: string;       // = commandText (pre-expansion)
  expandedText: string;       // CommandLineApi.format of the post-expansion parsed
  chain?: string[];           // multi-step expansion chain (length > 1)
}

interface CommandContext {
  // ...existing fields...
  aliasExpansion?: AliasExpansionInfo;
}
```

Controllers that branch on alias-vs-direct read `ctx.aliasExpansion`;
controllers that don't care never see it. The
`MudlogApilevel` auto-emit's structured payload
includes the same record under `payload.aliasExpansion` so log
consumers can correlate.

## Player command

`alias` is a single subcommanded verb. Mirrors the shape of
`settings`.

| Subcommand | Effect |
|---|---|
| `alias` / `alias list` | List effective aliases (defaults + persistent + session) with source markers (`*` for any override, `T` for tombstones) |
| `alias get <name>` | Show one alias's body and source |
| `alias set <name> <body>` | Validate + write. Body is greedy. Persistent unless `--session` is supplied |
| `alias set --session <name> <body>` | Same, but writes to the session store |
| `alias unset <name>` | Drop override (or tombstone a default — see §Concepts) |
| `alias describe <name>` | Show the entry with description, source |

Set-time validation rejects:

- Names that don't match `[A-Za-z_][A-Za-z0-9_-]*`.
- Bodies that fail to tokenize (malformed lex).
- Bodies with a pipe (multi-command).
- Bodies whose first token is flag-shaped (`--name=foo`, `-v`).
- Self-loop bodies (verb === alias name) without a positional ref —
  a self-loop with `$1` / `$@` is fine and recurses correctly.

The body's verb is NOT validated against the available command set
at set-time. The dispatcher surfaces "Unknown command: X" at
run-time the same way it does for raw input. This keeps
authoring cheap and lets the player author aliases that depend on
context-pushed verbs (an alias for `sit` works even when no Throne
is in the room — it just fails until one is).

The `--session` option is **subcommand-scoped** per the two-tier
option-scope rule in [command-routing.md](./command-routing.md).
`alias --session set foo bar` is rejected at the verb level;
`alias set --session foo bar` is the correct form.

## Persistence

`aliases` is a `Record<string, string | null>` and is listed in
`AliasMixin.persistentFields`. The Hydrator round-trips it by
reflection — no nested objects, no Sets, no special handling.
Tombstones (null values) round-trip through MongoDB cleanly as JSON
null.

`aliasesSession` is a plain `Record<string, string>` and is NOT in
`fieldMeta`'s persistent entries. Transient: dies with the in-memory host
instance. Reconnects to a still-live Avatar see the warm shell.

Tombstones never auto-cleanup. If a default is later removed from
the codebase, players' tombstones for it become harmless leftovers
(the resolver finds `null` for an undefined-default-name and treats
it as hard-set absence — `getAlias` returns `undefined`, the same
as for a name that was never declared).

## Composition order

`ShelledCharacter` is
`AuthorMixin(WorkspaceMixin(AliasMixin(EnvironmentMixin(FocusedMixin(Character)))))`.
AliasMixin sits mid-chain — WorkspaceMixin and AuthorMixin wrap it. It
has no method dependency on Environment or Focused state, but composing
it adjacent to the other shell-tier mixins in the prototype chain means
`MixinApi.queryMixins` returns the suite together.

## Future evolution

Out of scope for v1, deliberate follow-ups when demand surfaces:

- **Default-alias starter set.** AliasMixin ships with empty
  `static defaultAliases`. Content mixins are the right home for
  domain-specific defaults (a `MovementShortcutsMixin` shipping
  `n/s/e/w` etc.). The mechanism is in place.
- **Description round-tripping.** Persistent storage is body-only
  (`Record<string, string | null>`). A future "rich alias"
  persistence shape (`Record<string, string | null | { body, description }>`)
  could let players annotate their own aliases.
- **Subcommand-aware aliases.** Aliasing JUST a subcommand (e.g.
  `alias name = "set name"` triggering only under `player`) is
  out of scope; aliases replace whole verbs only. Subcommanded
  bodies are fine: `alias setname = 'player name'` produces a
  complete `player name <user-args>` command.
- **`$0` / `$#` / `$*`.** Dropped from v1; better modeled as
  synthetic vars or alias-frame state if added later.
- **Client-side schema delivery.** Aliases are not pushed onto the
  client's command schema. Future autocompletion may revisit.

## Cross-references

- [shell-environment.md](./shell-environment.md) — the substrate
  pattern AliasMixin parallels (two-store, schema-on-mixin / static
  defaults, cross-host helper). Read first if you haven't seen the
  pattern.
- [command-routing.md](./command-routing.md) — where alias
  expansion sits in the dispatch pipeline.
- [command-parsing.md](./command-parsing.md) — `CommandLineApi`,
  `RawToken`, `parsePipeline`, `format()` round-trip.
- [collections.md](./collections.md) — Shape B canonical surface
  the AliasMixin methods follow.
- [mixins.md](./mixins.md) — composition-driven static-data
  pattern.
- [../antipatterns.md](../antipatterns.md) § Reaching Into Raw
  Alias Storage — call-site rules.

## Antipatterns

| Don't | Do |
|---|---|
| `avatar.aliases['foo'] = 'look'` | `avatar.setAlias('foo', 'look', { actor: avatar })` |
| `avatar.aliases['foo'] = null` (manual tombstone) | `avatar.removeAlias('foo', avatar)` — tombstones automatically when the name has a default |
| Read from `avatar.aliases` directly | `avatar.getAlias(name)` (lookup chain) or `avatar.getAliases()` (resolved view) |
| Call `ShellApi.expandAliases` from a controller | Expansion is a substrate-pipeline concern; controllers read `context.aliasExpansion` if they need to know an alias fired |
| Skip set-time validation by writing the body string straight to `aliases` | Always go through `setAlias` so the body is tokenized once and rejected if malformed |

## Crossing into a sandbox circle

This state **forks onto the wire body** (fork-only — a change made
inside a circle discards with it). The vessel is a baseline body, and
"baseline" means no gear and no chattel, not a stranger's defaults: a
maker who has to retype full verbs, re-pick a theme, or re-open a
layout inside their own workshop is being punished for using it. See
[sandbox.md](./sandbox.md) § the crossing.
