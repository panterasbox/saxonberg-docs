# WorkspaceMixin

Per-character working position in the **content** and **source**
stores plus the navigation / I/O verb suite. Substrate for on-server
authoring — players and developers share the same capability
surface; only permissions (a future landing) gate authority.

The two trees:

- **content** — the authored game world. Path-shaped templates
  persisted in the `domain` Mongo collection, cloned at runtime
  into Stuff. Identifiers like `/platform/agent/Avatar/foo`.
- **source** — the engine source code. The filesystem under the
  project's `packages/`. Identifiers like
  `/server/src/mud/api/stuff.ts` (rooted at the sandbox root).

"Templates are code too" — the older `template` / `code` split was
lossy, so the substrate uses content / source.

Composition: applied to `ShelledCharacter` after `EnvironmentMixin`
and `AliasMixin`.

## State

Two cwds, persisted on the avatar:

- `contentCwd` — current location in the content tree
- `sourceCwd` — current location in the source tree

Default for both is `/`.

## Settings keyspace (`workspace.*`)

| Key | Type | Default | Purpose |
|---|---|---|---|
| `workspace.tree` | enum | `'content'` | Which tree workspace verbs default to. Values: `'content'` / `'source'` / `'mirror'`. Per-call `-c` / `-s` flags override. |
| `workspace.home` | string | `/` | Canonical `$HOME` path; one value across both trees. |
| `workspace.editor` | string | `''` | External-editor handoff target. Declared in v1; consumed by a future `edit` verb. |
| `workspace.pageSize` | number | `25` | Soft line cap honored by `cat` (and `ls -l`). 0 disables the cap. |

### `workspace.tree = 'mirror'`

Mirror mode keeps the two cwds in sync. Operationally:

- A `cd` (or any cwd write through `setCwdAuto`) with no explicit
  `-c` / `-s` flag updates **both** cwds. Single-tree flags break
  the mirror for that one call so the player can step out and back
  without re-syncing manually.
- Read operations (`ls`, `cat`, `grep`) default to the **content**
  tree — the path shape is identical in both trees and content is
  canonical. Pass `-s` to read from source instead.

The legacy one-shot `cd --mirror` flag remains as a per-call
"copy active cwd to the other tree" override, distinct from the
persistent setting mode.

## Synthetic vars

Read-only. The `var set $name` flow rejects these because synthetic
resolution wins over stored vars in `ShellApi.expandVariables`.

| Var | Reads |
|---|---|
| `$PWD` | active tree's cwd (per `workspace.tree`) |
| `$CPWD` | content tree's cwd |
| `$SPWD` | source tree's cwd |
| `$HOME` | `workspace.home` setting (resolved through `resolveSetting`) |

## Verbs

All verbs that pick a tree per call accept `-c` / `--content` and
`-s` / `--source` as the explicit override flags; with neither
flag, the `workspace.tree` setting decides.

Identifier alternation per principle 5 of the shell-tooling slate:
positional `<path>` for the path form, `--mql <expr>` for the MQL
form, mutually exclusive at the call site.

| Verb | Args | Notes |
|---|---|---|
| `pwd` | `-a` | Print active cwd (or both + tree-mode with `-a`). |
| `cd` | `<path>`, `--mql`, `-c`, `-s`, `--mirror` | Validates target is a Zone template (content) or directory (source). `--mirror` (one-shot) copies active cwd to the other tree. In `workspace.tree = 'mirror'`, an unflagged `cd` updates both cwds via `setCwdAuto`. |
| `ls` | `<path>?`, `--mql`, `-c`, `-s`, `-l`, `-R` | Glob-aware (`*`, `**`, `?`). Long form is one entry per line; `-R` recurses. |
| `cat` | `<path>`, `--mql`, `-c`, `-s` | Renders template `data` (folder → "use ls"); reads files. Soft cap from `workspace.pageSize`. |
| `grep` | `<pattern>`, `<path>?`, `-c`, `-s`, `-i`, `-r` | Line-oriented regex match. Output: `path:line: matched`. |
| `write` | `<path>`, `--mql`, `-c`, `-s`, `--class`, `--hydrator`, `payload.data`, `payload.body` | Honors `workspace.tree` like the read verbs when it picks a single tree (`content` / `source`). Under `mirror` mode the setting doesn't pick — writing the same content to both is nonsensical, so an explicit `-c` or `-s` is required. The two flags are mutually exclusive when both supplied. **Two payload fields, one per tree** (both payload-only, populated via `CommandApi.assembleFromStructured`, never typeable in the shell): `data` (`type: struct`) for content writes — IS the template's `data` map, passed straight to `TemplateApi.saveTemplate`; the controller validates against the resolved class's static `dataSchema` (JSON Schema fragment) when present. `body` (`type: string`) for source writes — raw file bytes. See [command-spec.md § payload](./command-spec.md) for the broader pattern. Content writes accept `--class` (default `/lib/stuff/Idea`) and `--hydrator` (default `/platform/idea/persistence/PersistentHydrator`; pass `--hydrator=` to omit). Source writes ignore both. |
| `mkdir` | `<path>`, `-c`, `-s` | Creates a Zone template (content) or directory (source). Folder/leaf invariant fires through the persistence chokepoint. |
| `rm` | `<path>?`, `--mql`, `-c`, `-s`, `-r` | No `-f` v1 — folder/leaf is a hard correctness invariant, not a bypassable policy. |
| `cp` | `<src>`, `<dst>`, `--mql`, `-c`, `-s` | Same-tree v1; cross-tree copy is a follow-up. |
| `mv` | `<src>`, `<dst>`, `--mql`, `-c`, `-s` | Content: clone-then-delete. Source: `fs.rename`. |

## `pickWorkspaceTree(giver, flags)`

Shared helper exported from `lib/shell/Workspace.ts`. Verbs thread
this so the precedence (`-s` > `-c` > `workspace.tree` setting >
`'content'` fallback) lives in one place.

`-s` wins over `-c` if a player somehow passes both — the louder
explicit signal.

## SourceTreeApi

`mud/api/source-tree.ts`. Wraps `node:fs/promises` with a fixed
sandbox root — the project's `packages/` directory. Every operation
resolves paths against that root and refuses any path that escapes
it (`SourceTreeSandboxError`).

Two path-resolution helpers:

- `resolvePath(cwd, input, opts?)` — sandboxed. Returns an absolute
  OS path; throws on escape. Used by source-tree operations.
- `joinLogical(cwd, input, opts?)` — pure logical-path normalisation.
  Returns an absolute path beginning with `/`, no sandbox check.
  Used for the content tree where paths are virtual identifiers,
  not OS file paths.

### The shipped-resource face (synchronous)

The async surface above backs the author shell. A second, **synchronous**
group serves a different caller: mudlib modules that need to load an
authored data file shipping in the source tree — a char-gen roster, a
theme catalogue, a command schema — from a static initialiser,
`postRegister`, or a lazy first-use path that cannot await.

- `readResource(moduleUrl, relativePath)` — raw text.
- `readYamlResource<T>(moduleUrl, relativePath)` — text + YAML parse.
- `readJsonResource<T>(moduleUrl, relativePath)` — text + JSON parse.
- `parseYaml<T>(text)` — the same parser for text already in hand (a
  command argument, a CMS field). No file involved.
- `toMudPath(absolute)` — absolute source path → `/`-rooted mud template
  path. Pure arithmetic, no I/O. The single definition of the mud root;
  `CommandDefinition` and `CommandLogic` used to keep one each.
- `resolveFrom(filePath, relativeRef)` — resolve a ref against the
  directory containing `filePath` (the sibling-file rule an authored
  spec uses to point at its controller).

`moduleUrl` is the caller's own `import.meta.url` — a **language
construct, not an import** — which is how a mudlib module names a file
without reaching outside `src/mud/`. The read itself happens here, in the
Api tier, and is sandbox-checked like everything else on this face. This
group exists because of [the import
boundary](../architecture.md); before it, five modules held their own
`readFileSync`.

## See also

- [shell-environment.md](./shell-environment.md) — settings keyspace
  + schema-on-mixin pattern this mixin's settings register through
- [shell-author.md](./shell-author.md) — sister mixin; lifecycle and
  code-execution verbs
- [perceiver.md](./perceiver.md) — sister mixin (the `scry` verb);
  at-a-distance perception
