# Scripting (v1 engine — the command-native interpreter)

The scripting language is a **real, command-native interpreter** — not a
macro recorder and not a second runtime. Its atoms are the same verbs a
player types at the prompt (`pour`, `say`, `go`), and every command a
script runs goes over the **real command bus** (resolve → validate →
execute), gated and attributed exactly as if hand-typed. A script is just
a *program made of commands*; the engine adds control flow, variables,
values, blocks, and time (coroutines) on top of that one atom.

It shares `EvalScript`'s **philosophy of isolation, not its runtime**.
`EvalScript` (`lib/script/EvalScript.ts`) is a `vm`-sandboxed *TypeScript*
surface for trusted authors; the interpreter here is a *command* surface
for everyone, sandboxed by **resource governance** and the **call-security
gate** rather than a VM. They are distinct tenants of the same
`lib/script/` directory — the physical encoding of that relationship.

Built by `docs/plans/scripting-mvp-plan.md` (P0–P10) over
`docs/requirements/scripting-mvp-requirements.md`. The integrating demo is
cocktail-making at Dave's Bar.

## What the engine gives you (the surface)

```
pour gin into mixing glass; stir mixing glass      # multi-statement
set $base (spirits where category = gin | first)   # a value, MQL island
if ($base) { pour $base into glass }               # control flow as commands
def martini ($brand) { pour $brand … ; strain … }  # named, parameterized
make martini with Crowsfoot                        # invoke a recipe-script
wait 5m; say "Last call."                           # coroutine — game-time
every 3m { say "Closing up." }                      # perpetual cadence
# this is a comment                                 # bash-rule line comments
```

Three syntactic facts do all the work: **statements** (separated by `;`
or newline), **blocks** (`{ … }`, a first-class value), and the
**`( )` island** (an expression evaluated by MQL + a tiny infix layer).
Everything else — `if`/`each`/`while`/`def`/`set`/`wait`/`every`/`when` —
is *spelled like a command* but evaluated by the interpreter.

## Grammar & the parser (P1)

The parser is a **pluggable `Parser`** (`lib/command/parsers/script.ts`),
selected by the `shell.parser` setting. It **wraps `msh`, it does not
replace it** — the load-bearing backward-compatibility decision:

- A bare command (no top-level separator, no standalone block) is handed
  **straight to `msh`** and returned as `{ parsed }`. This path is
  *literally msh*, so the parse is **byte-identical** to the old prompt
  (backed by a regression corpus). The default `shell.parser` flipped to
  `script` in P6 precisely because this guarantee makes the flip safe;
  `msh` stays selectable.
- Input with a top-level separator or a standalone `{ }` block (or a
  leading builtin verb) parses to a `Script` AST and returns
  `{ script }`.

The tokenizer is **recursive / delegating, never a flat re-tokenize**.
The only genuinely-new lexical work is statement separators, standalone
blocks, balanced `( )`/`{ }` slicing, `$name`, and `#` line comments.
Everything else is honored by **delegation**:

- **`|` stays reserved** — lexed into the pipe-shaped `Pipeline` AST node.
  v1 executes single-command pipelines only; the node is mandatory anyway
  (the forward-compat commitment, so the value→field binder is built
  general, not replay-specific).
- **`;` / newline separate statements.** A `;` inside `"…"` stays literal
  (the bash rule): `say "a; b"` is one command, `say a; b` is two.
- **`( )` delegates to MQL + infix.** A boundary `(` slices the balanced
  region and stores its **raw inner text** as an `Expr` node — *not*
  re-tokenized at the script level. The interpreter hands it to MQL / the
  infix evaluator at eval time.
- **`{ }` disambiguates by attachment.** A `{` at a token boundary is a
  standalone block; a `{` *glued* after a token (MQL's `coin:{5}`,
  `water:{2 cups}`) is consumed verbatim into the word. Disambiguation is
  strictly by the preceding character, **never by content-guessing**.
- **`$name` / `${name}`** become `VarRef` nodes (frame-first resolution,
  shell-fallback — below). `$` *extends* the shell variable namespace, it
  does not fork it.
- **`#` at a token boundary is a line comment** (the bash rule), skipped
  to end-of-line so authored `.script` files can be annotated. A glued `#`
  mid-word stays literal; comments never reach the AST, so `format()`
  drops them.

A block's inner text is **parsed once** into a `Script` fragment — the
anti-Tcl keystone: a block is never a re-scanned string.

### AST (`lib/script/ast.ts`)

`Script` → `Pipeline[]` → `Command[]` → `Arg[]`, where `Arg` is
`Literal | VarRef | Expr | Block`. The `Pipeline` node sits between script
and command even in v1 (single-stage), so piping is a later *execution*
feature, not an AST migration.

## The interpreter & execution model (P2 — the keystone)

The interpreter (`lib/script/Interpreter.ts`) is a **synchronous,
effect-yielding generator** — explicitly **not** a recursive `await`
tree-walk. This is the single highest-leverage decision in the build:
suspension (P5 coroutines) and preemption (resource governance) both
reduce to *"stop advancing the generator, persist the position"*.
Retrofitting suspension onto a recursive interpreter would be a rewrite,
so the generator shape is committed in P2, not deferred.

`eval*` methods are generators that `yield` an **`Effect`**:

- `dispatch` — run a bound command `{ command, model, prep }` over the
  bus. The interpreter renders the command line, asks `CommandApi` to
  assemble it to a `{ command, model }`, and yields it; the pump calls
  `CommandGiver._dispatchBound` (the bound tail: resolve → validate →
  execute). The interpreter **conducts the bus, it never bypasses it**.
- `slice` — the preemption yield (below).
- `suspend` — a coroutine `wait` / `every` / `when` (P5).

A **pump** (`drive` / the `Coroutine`) advances the generator, handling
each effect, and is the only async layer. The eval core stays sync, which
is exactly what makes it suspendable.

### Scope, frames & `$` (P2)

`Scope` (`lib/script/Scope.ts`) is the lexical environment: a frame chain
with `define` / `get` / `has` / `assignExisting`, `child()` for nested
blocks. `$x` resolves **frame-first, shell-fallback**: the script frame
first, then a miss delegates to `ShellApi.expandVariables` (so synthetic
shell vars like `$focus` still resolve). `set $x …` writes the frame.

### The block keystone & `( )` island

A **`Block`** (`lib/script/Block.ts`) is a first-class value-object: a
parsed `Script` body + a captured `Scope` (a closure). It is inert — it
carries no execution logic; the interpreter invokes it (`invokeBlock`).
`if`/`each`/`while`/`def`/`every`/`when` all take block args **lazily**
(unevaluated), which is why builtins are interpreter-intrinsic rather than
bus verbs (a bus verb would eagerly evaluate its arguments).

The **`( )` island** is the expression sublanguage. The raw inner text
routes to **MQL** (`MqlApi.resolveOne/Many`) for set-queries, wrapped by a
**deliberately minimal infix layer** (`Expression.ts`): comparison
(`== != < > <= >=`), `and`/`or`/`not`, and emptiness. No precedence
parser, no operator zoo — set-queries pass through to MQL unchanged. The
one **uniform value model** (`Value.ts`: `string | number | boolean |
Stuff | StuffList | Block | void`) is shared by `set` / `$` / `( )` /
block-yield, with **MQL-set-emptiness-as-falsiness** centralized in
`isTruthy` — the commitment that makes piping values into fields uniform.

### Control flow as commands (P3)

`if`/`each`/`while`/`def`/`set` are **interpreter-intrinsic**: spelled
like commands, branched inside `evalCommand` *before* `bindCommand`, taking
raw block/condition args. `each` binds `$it` per item. `def name ($p) {…}`
uses **simple named positional params** (no implicit `it`, no rest /
defaults) and registers a `ScriptDef` (params + body + captured scope).

## Coroutines: scripting in game-time (P5)

A script that says `wait` must not block the prompt. The **`Coroutine`**
(`lib/script/Coroutine.ts`) pumps the generator across **game-clock**
frames and **detaches** on suspend — the prompt returns immediately, the
run resumes later. Four shapes:

- **`wait <dur>`** — one-shot: resume after the duration (`5m`/`30s`/`2h`/
  `1d`/bare-seconds), scheduled on `WorldClockApi.after` (**game-time, never
  a bare `setTimeout`**).
- **await-engaged** — *not* a yielded effect: the pump detects, after a
  dispatch, that the command started a **timed `EngagedMixin` activity**
  and waits for that activity to complete before advancing. This is the
  pacing rule that makes a hand build trickle out step-by-step (the P4
  crafting verbs are the signal — no new events).
- **`every <dur> { block }`** — perpetual cadence; the main run does not
  resume past it.
- **`when (cond) { block }`** — poll the condition; run the block once it's
  truthy, then resume.

> **Game-clock scheduling exists.** `WorldClockApi.after/every/at` return
> cancellable handles riding game-time (`SchedulerApi`'s "layer on top"
> note is stale). The coroutine substrate uses it directly — no adapter.

**Cancellation** (`ScriptAbortReason`): a detached run is registered
per-actor in `ScriptLogic`; `stop`/`cancel` (the barge-in) reaches it via
`ScriptApi.cancelAll`, which aborts an in-flight engaged step so its effect
**never lands** (partial matter stands; no rollback). `every`/`when`
sub-coroutines cancel with the parent.

## Resource governance (the sandbox)

Isolation is **hard**, and every limit is an `AppSettings` key
(`script.*`), tiered by **authorship**:

- a **universal preemption slice** — every `sliceSteps` steps the
  interpreter yields a `slice` effect (`ScheduleApi.schedule(0, …)`), so no
  script monopolizes the event loop;
- a **total ceiling** — `maxSteps` / `maxDispatch` / `maxDepth`, exceeding
  which raises a `ResourceLimitError`.

`ScriptLogic.resolveLimits(authorPath?)` picks the tier: **player-home /
inline** scripts get the tight budget; **released platform content**
(`/obj/`, `/domain/`) the large one. Authorship, not the caller, sets the
ceiling.

## The two player surfaces (P6)

1. **The prompt is the interpreter.** `shell.parser` defaults to `script`;
   a bare command is still msh (byte-identical), but a multi-statement or
   block line runs as a script. This is how a player gets loops and
   variables with zero ceremony.
2. **Named scripts.** A session `def` persists per-actor (`SESSION_DEFS` in
   `ScriptLogic`, the interpreter's `defs` map) — a `def` typed at one
   prompt is invocable at the next via `make <recipe> [with <arg>]`
   (`MakeController` → `ScriptApi.invoke`, binding the first param from
   `with`). Authored recipe-scripts persist in the store (below).

## The store: scripts as a document `kind` (P7)

Scripts are **source text**, not Stuff templates (a template hydrates a
Stuff; a script hydrates nothing) and not git-deployed code. They persist
as **one `kind` of stored document** in the generic, owner-claimed
**document store** — the third path-addressed tree (source / template /
**document**), its own subsystem ([document-store.md](./document-store.md)).
A script is a `StoredDocument` with `kind: 'script'`, `data: { source }`,
keyed on the path (scope encoded):

```
/home/<player>/scripts/<name>     personal (recorded or written)
/domain/<world>/scripts/<name>    managed world content
```

The stored `data.source` is the canonical text (what the CMS shows),
**re-parsed on resolution** — no compilation, which is why scripts reach
the runtime-writable tree first. The **split of concerns** is the point:

- **`DocumentApi`** owns *storage* — `read` / `list` / `save` (find-or-
  create), the owner-access gate (an owner owns their own `/home/<self>/`
  branch, else zone / slice-walk), the owner stamp from execution context
  (`getActingAuthor`, never a parameter), and `recordAuthoring` keyed on
  the path (`DocumentLogic` is a named authoring transport in the
  broadened gate). It is **kind-agnostic** — never inspects `data`.
- **`ScriptLogic`** owns the script *semantics* — `saveScript` (→
  `DocumentApi.save(path, 'script', { source })` + invalidate the AST
  cache), `resolveScript` (→ `DocumentApi.read` → parse `data.source` →
  the per-path AST cache the generic store deliberately doesn't keep),
  `invokeByPath`, and `goLive` (the script-specific cache invalidation).

### The CMS third tree

`CmsBackend` is `content | source | document`; `CmsLogic` dispatches the
unified tree / read / stat / write per backend. The document backend is
generic — the record's **`kind`** drives the editor treatment: a `script`
kind is a plain-text code leaf whose write funnels through
`ScriptApi.saveScript` (the script chokepoint: gate + provenance + AST
go-live), and any other kind is its `data` as JSON. So authored scripts
edit-and-go-live in the same explorer as templates and source — and the
tree is ready for the next owned-JSON kind (dorm customization) without a
new backend.

## Demonstration capture (P8)

The **third authoring source** (after typed-inline and authored-as-content):
a hand build *becomes* a script. As a player builds a drink by hand, the
intermediate vessel records each completed step's verbatim command source
(the **vessel-as-buffer** capture boundary — a *scripted* build records
nothing, the natural manual-vs-scripted discriminator). At the terminal
`strain`, the **`Transcriber`** (`lib/script/Transcriber.ts`):

1. parses the joined step sources via the `format()` round-trip (P1) —
   so the artifact is **inspectable + editable source**, not an opaque
   macro;
2. **lifts the base spirit** (the first step's input) to a single
   `$brand` param;
3. wraps it in `def <recipe> ($brand) { … }`;
4. banks it to the builder's home path in the store.

Learning a recipe *is* getting the script.

## The chronicle knowledge ladder (P9)

`make <recipe>` is gated on a per-character learning ladder, **derived on
read** from the chronicle ledger (the renown precedent: a dumb store, a
smart consumer — no stored set, no mixin). `RecipeKnowledge`
(`lib/script/RecipeKnowledge.ts`):

```
unknown → known-of (a chronicle claim, on reading the menu)
        → can-make (a chronicle deed, on the first faithful hand build)
```

Reading the menu mints the **known-of** claim (idempotent). The first
faithful manual build mints the **can-make** deed **and** transcribes the
recipe-script — the *same act*. `make <catalogue-recipe>` declines until
you can-make it; a player's own non-catalogue `def` is ungated (you wrote
it). Claim and deed use **distinct keys** (`recipe-known:` /
`recipe-made:`) because `ChronicleApi.recordOnce` dedups on `{owner, key}`
regardless of kind. `ScriptLogic.invoke` auto-loads a learned home
recipe-script by name, so `make martini` finds the banked script after the
first build.

## Authored demo content (P10)

`mud/domain/lounge/scripts/` holds the authored exemplars — all **content**
over general substrate, **no bar-specific engine class**:

- `martini.script` / `daiquiri.script` — parameterized recipe-scripts
  (`def <name> ($brand) { … }`), the authored-as-content twin of what the
  Transcriber emits;
- `last-call.script` — the closing-time coroutine (the shift-change
  exemplar): `say` → `wait 5m` → `say` → a perpetual `every` reminder.

`ScriptSeeder` (`backend/ScriptSeeder.ts`, the `RecipeSeeder` precedent)
banks each `.script` at `/domain/lounge/scripts/<name>` at boot,
idempotently — the live wire. These world scripts are the **`invokeByPath`**
targets (an operator beat or an NPC bartender's brain); they are
deliberately **not** wired into a player's `make` (that resolves the
player's own session `def` or home recipe-script — the demonstration
ladder's output).

## Relationship to the command pipeline & EvalScript

- **The command pipeline** is the substrate, not a peer. A script is a
  program whose instructions are commands; `dispatch` is `_dispatchBound`,
  the same bound tail a parsed prompt command takes. There is no second
  execution path, no second security model — **gated, attributed,
  re-resolved**, every statement.
- **`EvalScript`** is the contrasting surface: trusted TypeScript in a
  `vm` sandbox for authors who need to reach the raw object model. The
  command interpreter is the everyone-surface: untrusted, command-only,
  sandboxed by resource governance + the call-security gate. Same
  *philosophy of isolation*, different *runtime* — and the reason both
  live in `lib/script/`.

## File map

| Concern | File |
|---|---|
| Value model + truthiness | `lib/script/Value.ts` |
| AST node types | `lib/script/ast.ts` |
| Block keystone (closure) | `lib/script/Block.ts` |
| Lexical scope / frames | `lib/script/Scope.ts` |
| `( )` infix evaluator | `lib/script/Expression.ts` |
| The generator interpreter | `lib/script/Interpreter.ts` |
| Builtin verb set | `lib/script/builtins.ts` |
| Coroutine pump + cancellation | `lib/script/Coroutine.ts` |
| Abort reasons | `lib/script/AbortReason.ts` |
| The wrapping parser | `lib/command/parsers/script.ts` |
| Document store (scripts ride it) | `lib/document/StoredDocument.ts` + `api/document.ts` + `obj/api/DocumentLogic.ts` ([document-store.md](./document-store.md)) |
| Demonstration capture | `lib/script/Transcriber.ts` |
| Recipe knowledge ladder | `lib/script/RecipeKnowledge.ts` |
| Api façade | `api/script.ts` |
| Logic singleton (`/obj/api/script`) | `obj/api/ScriptLogic.ts` |
| `make` verb | `obj/command/crafting/MakeController.ts` + `cmd/crafting/make.yaml` |
| Boot seeder | `backend/ScriptSeeder.ts` |
| Authored demo scripts | `mud/domain/lounge/scripts/*.script` |

## `ScriptApi.mintEvalScratch`

The `eval` verb's scratch singleton (`<jurisdiction>/_eval`) is minted
here rather than in the controller. The mint stamps an identity path so
MQL's path atom can address the scratch and a later bare `eval` can
re-run it — and `Stuff.setTemplatePath` is `ApiOnly`-gated, so a
controller doing it itself dies on the gate. It did: **every**
`eval <code>` was failing on that gate, in the field as well as inside
a circle, until the sandbox build's live pass caught it. See
[sandbox.md](./sandbox.md) for the jurisdiction half.

## The eval sandbox and the import boundary

`EvalScript` (`lib/script/EvalScript.ts`) is the wizard-gated eval
container. Node's `vm` is a capability, so under [the import
boundary](../architecture.md) it may not be imported from the mudlib —
the three vm calls (`new Script`, `createContext`, `runInContext`) live
in `ScriptLogic`, behind `ScriptApi.compileSandboxed` /
`ScriptApi.runSandboxed` and an opaque `CompiledSandbox` handle.

What did **not** move is the part that matters for review: the
`SANDBOX_NAMES` allowlist and the `self`/`target` receiver bindings stay
in `EvalScript`. Deciding what an eval'd script can see is mudlib policy;
assembling that plain object needs no privilege. A reader auditing the
sandbox surface still reads one file, and the planned `isolated-vm`
migration still replaces one function plus its Api counterpart.
