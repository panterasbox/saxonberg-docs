# Shell Environment

`EnvironmentMixin` is the per-instance settings keyspace — schema-
validated values that features read at runtime to configure their
behavior, and that players mutate through the `settings` and `var`
commands. It's the substrate of the **shell subsystem** at
`lib/shell/`, named for its bash/zsh inspiration; future shell mixins
(prompt, history, working directory, aliases) will sit alongside it
under the same folder.

The shape lives in `lib/shell/Environment.ts`. Today, only `Avatar`
composes it.

## Core Principle

Settings live on the mixin that owns the concept they describe. The
substrate (`EnvironmentMixin`) declares no settings of its own —
`MobileMixin` declares the movement-message templates, future
`PromptMixin` will declare the prompt format, and so on. Adding a
mixin to a host pulls its settings in for free; removing it pulls
them back out. **There is no central registry.**

A setting that "every avatar needs" is a statement about the *feature*
that needs it: introduce a feature mixin and have every avatar compose
it. Don't pile general-purpose settings onto the substrate.

## Concepts

### Two stores

Each host carries two stores per instance:

- **`persistentStore`** — schema-declared overrides whose entry has
  `lifetime: 'persistent'` (the default). Saved through the `Hydrator`
  like any other persistent field. Survives logout and reconnect.
- **`sessionStore`** — schema-declared overrides whose entry has
  `lifetime: 'session'`, plus all ad-hoc `var` writes. Transient: not
  persisted, lives only for the lifetime of the in-memory host
  instance. Avatars survive reconnects (the in-memory instance
  outlives the websocket), so the session store is a "warm shell"
  across short disconnections; it dies when the instance is
  destroyed.

`setSetting` routes by the entry's `lifetime`. `setVar` always writes
to `sessionStore`.

### Schema-on-mixin

Each mixin declares its keys via a static field:

```ts
class FeatureMixin {
  static settings: SettingsSchemaEntry[] = [
    {
      key: 'feature.greeting',
      type: SettingTypes.String,
      default: 'hello',
      description: 'spoken on /wave',
    },
    // ...
  ];
}
```

That static is the **only** place a setting is declared. No
`register()` call at module load, no shared registry, no global
"unowned" partition. The host's effective schema is computed on
demand by walking the host's full prototype chain and unioning each
layer's `static settings`. A key declared on two layers throws —
defensive, surfaces collisions early.

The walk is unmemoized today. If consumer hot-paths grow, memoize
keyed on the constructor (it's the only input that varies for a given
host class).

**Schema-on-owner generalization.** The walk picks up `static settings`
from both mixin layers AND substrate classes. The schema-on-mixin
pattern generalizes to schema-on-owner: each setting lives wherever
the concept lives. The canonical class-level example is
`Avatar.settings`, which declares `world.autosave.interval` — the
cadence of the Avatar persist-back periodic backstop. The autosave
concept is owned by the Avatar lifecycle substrate; declaring on the
substrate class keeps "the concept's owner declares it" intact
without inventing a one-off mixin. When a second persist-back
consumer materializes (a future persistence/persist-back mixin),
the setting moves to that mixin and Avatar composes it —
same shape, different layer.

### Lookup chain

`getSetting<T>(key)`:

1. Walk schema. If the key isn't declared, return `undefined`.
2. Read the entry's `lifetime`-specified store; return the override
   when present.
3. Fall back to `entry.default`.

`setSetting<T>(key, value, actor)`:

1. Walk schema. Unknown key throws — `var` is the ad-hoc path.
2. Type-shape check (`SettingsSchemaEntry.type`). Wrong-type values
   throw before the custom validator runs.
3. Optional `validator(value)` callback. Returns `true` on success or
   an error message string.
4. Privacy check (see below).
5. Write to the entry's `lifetime`-specified store.

### Privacy

A schema entry may declare `private: true`. Private entries are
writable only when the actor performing the write is reference-equal
to the host instance. The check is `actor !== this` — type identity
doesn't matter; reference does. Reads bypass privacy entirely.

**Privacy gates writes only.** It's a hatch for entries where
programmatic mass-rewrite from other features would be undesirable
(future "guest mode" pre-set, admin-imposed templates, etc.). Use it
when the threat is concrete; default to `private: false` (omit the
field) otherwise.

The `actor` argument is typed `Stuff`, not `Avatar`. The check is
reference equality, so the runtime type of the host doesn't matter.
Typing it `Avatar` would create an import cycle (`lib/shell` →
`obj/Avatar` → `lib/character/Character` → … → `lib/shell`).

> **Known weakness, future tightening.** Once command stacking is
> implemented, `actor === target` is spoofable by a wrapper command
> that re-points the actor. The check should tighten to: "the
> *interactive originator* of the current command stack is the host,
> AND the stack contains exactly one command frame." Both pieces come
> from `ExecutionContextApi`. Carry this gap forward in any successor
> doc.

### Settings vs vars

Two flavors of write, two surfaces:

- **Settings** are schema-declared, type-validated, route to the
  store dictated by `lifetime`, may be private. The player command is
  `settings`.
- **Vars** are ad-hoc strings in `sessionStore`. No schema, no
  validation, no privacy. The player command is `var`.

`setVar` rejects names that match a declared schema key — closes the
back door where `var` would otherwise let a player bypass schema
validation by clobbering a declared key in the session store. Players
must use `settings set` for declared keys.

## API

```typescript
interface Environment {
  // Schema-validated settings.
  getSetting<T>(key: string): T | undefined;
  setSetting<T>(key: string, value: T, actor: Stuff): void;
  unsetSetting(key: string, actor: Stuff): void;
  listSettings(): SettingsSnapshotEntry[];
  describeSetting(key: string): SettingsSchemaEntry | undefined;

  // Ad-hoc session vars.
  setVar(name: string, value: string): void;
  unsetVar(name: string): void;
  listVars(): Record<string, string>;
}

interface SettingsSchemaEntry<T = unknown> {
  key: string;
  type: SettingType;            // SettingTypes.{String,Number,Boolean,Enum,Struct,List}
  default: T;
  description: string;
  lifetime?: 'persistent' | 'session';   // default 'persistent'
  private?: boolean;                       // default false
  enumValues?: T[];             // required when type === SettingTypes.Enum
  validator?: (value: T) => true | string;
}

interface SettingsSnapshotEntry {
  schema: SettingsSchemaEntry;
  currentValue: unknown;        // override, or schema default
  isOverridden: boolean;        // value present in lifetime-specified store
  sourceMixin: string;          // _mixinName of the declaring layer
}
```

Decoration: `setSetting` and `unsetSetting` are `@Unshadowable`. A
shadow that masked the privacy check would be a security hole. No
`@Final` (subclasses for tracing or extension are reasonable) and no
`@CallSecurity` (no callsite restriction).

`SettingTypes` is a constant table mirroring the `PropOperations`
pattern in `lib/stuff/Propertied.ts`:

```ts
export const SettingTypes = {
  String: 'string', Number: 'number', Boolean: 'boolean',
  Enum: 'enum',     Struct: 'struct', List: 'list',
} as const;
```

Declare schema entries with `type: SettingTypes.String` rather than
the bare literal — same convention as `PropOperations`.

## Cross-host resolution

Some settings are declared on mixins composed by hosts that don't
also compose `EnvironmentMixin`. `MobileMixin`'s movement messages
are the canonical case: present on both Avatars (which have
`EnvironmentMixin`) and NPCs (which don't). To keep the host-type
branch out of every consumer, the subsystem exports a free function:

```ts
resolveSetting<T>(host: Stuff, key: string, factor?: FormFactor): T | undefined;
```

- If `host` composes `EnvironmentMixin`, delegates to
  `host.getSetting<T>(key)` — full lookup chain, override-aware.
- Otherwise walks `host`'s full prototype chain (mixin layers and
  substrate classes alike), finds the declaring layer's schema
  entry, and returns its `default`.
- Returns `undefined` when the key isn't declared anywhere on the
  host's chain.

### ⭐ The per-form-factor rung

An entry may declare `perFactor: true`. That is **one key with an
OPTIONAL override**, not two mandatory keys — a player who wants the
same behaviour everywhere sets one value, and two independent keys
guarantee eventual silent drift.

Three rungs, and the order is the whole feature:

1. the stored override at `<key>.<factor>`, when a factor is named;
2. the stored override at `<key>`;
3. the schema default.

`setSetting` accepts a suffixed key **only** for a `perFactor` entry and
only for `desktop` / `mobile`; `findSchema` resolves the suffix back to
the base entry, so `shell.interpolate-vars.mobile` refuses as *no such
setting*. That is what keeps this an optional override rather than an
open namespace. The value is stored under the key as WRITTEN, beside
the base rather than replacing it.

⚠ **Rung 1 reads `getOwnSetting`, not `getSetting`.** `getSetting` falls
back to the schema default, so a suffixed read would ALWAYS return
something and rung 2 could never be reached — the override would
silently become mandatory, which is exactly the shape this refused.

⚠⚠ **This does not break the no-`cockpit.formFactor` rule.** That key
was never built because *the server cannot know a viewport, so such a
key would be a fake fact*. Two stored PREFERENCES assert nothing about
which is in force: the server resolves both and ships both, and the
client — which genuinely knows its own width — picks. Same split as
`cockpit.shelf`.

`shell.result` is the first consumer: `card` (default) · `terminal` ·
`both`, deciding whether a structured command result appears on a card,
in the transcript, or in both. See
[card-surface.md](./card-surface.md).

### ⭐ The per-KIND rung — the same shape, a second axis

An entry may instead declare `perKind: true`, which resolves
`<key>.<CardId>` by exactly the three rungs above. `cards.window` is
its consumer: `cards.window 600` is how long an unpinned card stays,
and `cards.window.subject 3600` keeps the things you have looked at for
an hour while everything else ages normally.

⚠ **Deliberately a second flag rather than a general `perSuffix`.**
What makes either of these an *optional override* rather than an open
namespace is that the suffix set is CLOSED and validated — form factors
for one, `CARD_IDS` for the other. A generalized mechanism would have
to accept any suffix, which is the open namespace both were written to
refuse. Two small closed rungs beat one that cannot say no.

This is **not** a `SettingsApi` — there's no Api boundary, no
registry, no proxy. It's a colocated helper in the same file as the
mixin. Callers that need defensive handling for undeclared keys can
`?? fallback` the result.

## Player commands

Two top-level commands, standard YAML view + controller pattern.

`settings` — schema-validated persistent store.

| Subcommand | Effect |
|---|---|
| `settings` / `settings list` | Snapshot grouped by source mixin, with `*` marking overrides. |
| `settings get <key>` | Show one value. |
| `settings set <key> <value>` | Validate + write. Coerces YAML's string input to the declared type. |
| `settings unset <key>` | Drop the override; default reapplies. |
| `settings describe <key>` | Schema entry: type, default, lifetime, description. |

`var` — ad-hoc shell variables in the session store.

| Subcommand | Effect |
|---|---|
| `var` / `var list` | List session vars (declared keys filtered out). |
| `var set <name> <value>` | Write to session store. Rejects declared keys. |
| `var unset <name>` | Drop from session store. |

In the player path, `actor === target === avatar` (the
`commandGiver`), so the privacy check passes by construction. The
check earns its keep against programmatic writes from feature code.

YAML coercion: `SettingsController.executeSet` coerces the string
value to the declared `SettingTypes.X`. Numbers fail loudly on
non-numeric input; booleans accept `true`/`yes`/`1` and
`false`/`no`/`0`; enums pass through to schema validation; `Struct`
and `List` are rejected at the command surface (no structured-value
syntax yet).

## Persistence

`persistentStore` is a plain `Record<string, unknown>` listed in
`persistentStore: { persistent: true }` in `fieldMeta`. The `?` modifier and
`= {}` initializer match the legacy-tolerant pattern from
`Propertied.savedProps` — existing avatar docs without the field
hydrate cleanly (the Hydrator skips fields not present in `data`,
the constructor default fills in).

`sessionStore` is plain — not in `fieldMeta`'s persistent entries — so the Hydrator
ignores it. No special clear hook on disconnect: the store dies when
the host instance does. Reconnects to a still-live Avatar see the
warm shell.

## Movement-message migration — canonical example

`MobileMixin` declares 10 settings under `messages.movement.*` —
schema-defaulted, player-overridable, **not private** (display-only
prefs, no concrete threat to gate against). The default body
factories `defaultDepartureSelf` etc. resolve via `resolveSetting`,
so Avatars see overrides through `getSetting` while NPCs render at
the schema default through the non-Environment fallback. Same code
path either way.

Override precedence inside `MobileMixin` (highest first):

1. `Exit.messageOut` / `Exit.messageIn` — room-author per-exit strings.
2. Source / destination room hooks — `getDepartureMessage`,
   `getArrivalMessage`, `getTeleportOutMessage`,
   `getTeleportInMessage`.
3. `messages.movement.*` settings — the player-overridable defaults.

Defaults are Liquid templates rendered by `ProseApi`; conditionals
inside the template handle the directional/bland split, so a single
`arriveSelf` / `arrivePeers` key covers both cases. See
[prose.md](./prose.md) for the templating language and helpers.

## Not settings: client state

`EnvironmentMixin` is for **player-tunable knobs** the player
configures via the `settings` shell command. UI state the cockpit
manages through its own gestures (tabs, theme, notification prefs,
keybinds, saved MQL queries, …) lives on `HasInteractiveMixin`'s
`_clientState` surface — see
[connection.md § Client state](./connection.md#client-state) for
the storage, schema, methods, and wire path. Per
[[feedback-settings-vs-propertied-vs-client-state]], do not reach
for the settings command surface to persist UI-driven state.

## Future neighbors

Planned shell mixins not built today, but whose shape constrains
decisions here:

- **`AliasMixin`** — shipped. Per-character verb aliases with the
  same two-store pattern (persistent + session), plus a defaults tier
  walked from `static defaultAliases` on any composed mixin and a
  tombstone mechanism for default-suppression. Body-side positional
  substitution (`$1`..`$9`, `$@`) lives in `ShellApi.expandAliases`.
  See [shell-alias.md](./shell-alias.md).
- **`HistoryMixin`** — command history, recall, search.
- **`PromptMixin`** — prompt rendering. `prompt.format` is the
  canonical example of "shell meta-config" — it lives on
  `PromptMixin` rather than `EnvironmentMixin` so the substrate
  discipline above doesn't have an exception.
- **`WorkingDirectoryMixin`** — cwd over the in-game filesystem.

If the per-avatar + two-store + schema-on-mixin pattern proves to be
genuine shared infrastructure, factor it out at the second-mixin
boundary, not preemptively.

## Variable interpolation

`$name` and `${name}` references inside command-line text expand
to the resolved value at matcher time. Implementation lives on
`ShellApi` (`api/shell.ts`) — the substrate Api for the msh shell
suite. The matcher (`CommandApi.assemble`) calls
`ShellApi.expandVariables` per `WordToken` value before binding to
a positional. Greedy slices have each token expanded before
rejoining; quoted slices expand the same way (one uniform rule —
shell-quoting is just token-grouping).

`$$` is left intact for MQL's last-result token to handle. Names
match `[A-Za-z_][A-Za-z0-9_]*` — no dots, no hyphens, matching the
existing setting/var naming convention.

> **Pronoun words are not shell vars.** `me`, `here`,
> `it`/`him`/`her`/`them` are first-class MQL keywords recognized
> by the resolver. Typing `look here` or `look him` works because
> MQL's pronoun seed handles them, not because of variable
> expansion. There is no `$me` / `$here` / `$it` alias.

### Synthetic vs stored vars

Two sources, with synthetic winning on collision:

- **Synthetic** — read-only, declared by mixins via static
  `syntheticVars: SyntheticVarEntry[]`. Same composition-driven
  pattern as `static settings`, `static commandContributions`,
  a `persistent` entry in `fieldMeta`. `ShellApi.lookupSyntheticVar` walks
  the giver's mixin chain on each expansion; first match wins.

  v1 ships exactly one entry: **`$focus`** on `FocusedMixin`,
  reading the live drilled fragment via `getFocus()`. NPCs
  without `FocusedMixin` get null on lookup.

  Naming note: **scope** is the general term for the per-resolution
  MQL search anchor; **focus** is the specific persisted state on
  the command giver. The synthetic var is named after the state it
  reads, not the role it plays.

- **Stored** — settable via `var set NAME VALUE`, read on
  expansion via `giver.listVars()`. Only available when the giver
  composes `EnvironmentMixin`.

Synthetic precedence: if a player does `var set focus foo`,
`$focus` still resolves to the giver's actual current focus.
Synthetic names are documented and stable; an accidental
collision is better surfaced as documented-name-wins than as
silent override.

### Gating

The matcher gates `expandVariables` per giver. `ShellApi` itself
is unconditional — direct callers (the YAML scope expander) want
to expand regardless of player settings. The two matcher gates:

- Per-host setting: `shell.interpolate-vars: boolean` (default
  `true`). Off → matcher's per-token pass skips expansion
  entirely; literal `$X` flows through.
- Mixin absence: givers without `EnvironmentMixin` skip expansion
  on the matcher path. Scripts pass literal MQL.

The YAML-side `scope:` expander always runs, since the YAML is
authored content rather than player input.

NPC scripts that hit `$X` either fail loud at MQL parse time
(`$` isn't valid MQL syntax outside `$$`) or pass a literal token
the resolver doesn't recognize. The contract: opt in to the shell
suite (compose `ShelledCharacter` instead of `Character`) if you
want drill state or var interpolation; the default NPC has
neither.

### Empty / missing variables

- Synthetic that returns empty (e.g., `$focus` with empty focus —
  shouldn't happen, focus defaults to `"here"`): the token
  becomes empty and drops out of the bind.
- Stored-var miss: empty substitution + soft-warn via `MudlogApi`
  ("unknown variable: $foo") when the giver is a Sensor. Failing
  the command would break scripts mid-flight; empty-substitute
  keeps things moving and surfaces the typo.

### YAML scope[]

`FieldDefinition.scope` accepts `string | string[]` in the YAML /
spec record. After `CommandDefinition.normaliseShape` runs, the
runtime value is always `string[] | undefined` — downstream code
treats it as an array, no `Array.isArray` checks needed. Each
entry runs through `ShellApi.expandVariables` (so `$focus` and
stored vars resolve at bind time) and is tried in order; first
non-empty result wins. The array form is the explicit fallback
chain — a verb that wants drill-first semantics declares
`scope: ['$focus', 'inventory, here']`. There is no implicit
"player focus tries first" rule; the YAML is authoritative, and
the help system can read the array to tell players which commands
respect drill.

### Defaults

`FieldDefinition.default?: string` lets a YAML field declare
fill-in text. The matcher's `bindPositionals` extends boundary-
lookahead to non-greedy fields too: if the next available token
belongs to a *later* field's `prepositions:` list, the current
field has no input — apply the default (or fail when required
without default). The default runs through `expandVariables`, so
`default: "$focus"` resolves at bind time. `required: true` +
`default:` is allowed; the default replaces the missing input.

The canonical use is `look.yaml` — `default: "$focus"` makes bare
`look` mean "look at what I'm focused on" without controller-side
special branches.

### Future evolution

When alias / function frame-local scope arrives, the lookup chain
extends — but does not replace — what's described here:

> frame-local (alias / function args)
> → synthetic (mixin-declared)
> → session store
> → persistent store
> → schema default (or, eventually, a `defaultCompute(host)`)

`EnvironmentMixin` owns session and persistent only. Frame-local
scope is the future alias / function executor's concern. The
scripting language and the interactive shell share session and
persistent stores (bash model); locally-scoped script variables,
if needed, arrive as a `local` modifier — they don't reclaim the
word `var`.

## Antipatterns

See [antipatterns.md](../antipatterns.md). The shell-environment
specific entries:

| Don't | Do |
|---|---|
| Declare a setting on `EnvironmentMixin` because "every avatar needs it" | Declare on the feature mixin that owns the concept; ensure every avatar composes it |
| Reach into `persistentStore` / `sessionStore` directly to bypass the privacy check | Always go through `setSetting(key, value, actor)` |
| Read a setting via bare property access on a store map | Use `getSetting` (or `resolveSetting` for cross-host) so the lookup chain and schema are honored |
| Use `var` for state that should be persistent | Declare a schema entry on the right mixin and use `settings` |
| Build a `SettingsApi` (or `SettingsRegistry`) wrapping `EnvironmentMixin` | Settings are mixin-owned; schema is static class data; a wrapping layer adds nothing |
| Branch on `MixinApi.isEnvironment(host)` at the call site | Use `resolveSetting(host, key)`; the branch is centralized |

## Crossing into a sandbox circle

This state **forks onto the wire body** (fork-only — a change made
inside a circle discards with it). The vessel is a baseline body, and
"baseline" means no gear and no chattel, not a stranger's defaults: a
maker who has to retype full verbs, re-pick a theme, or re-open a
layout inside their own workshop is being punished for using it. See
[sandbox.md](./sandbox.md) § the crossing.
