# AuthorMixin

Object-lifecycle and code-execution verbs on `ShelledCharacter`.

The mixin owns no instance state v1 — its value is the verb
contributions and a session-lifetime settings keyspace. Per-target
refusal lives on the targets themselves (witness-shape `canX`
hooks); the actor-side capability seam is handled by the verb
controllers and the future permission framework.

Composition: applied to `ShelledCharacter` after `WorkspaceMixin`
and `AliasMixin`.

## Verbs

All five verbs that accept an object identifier follow principle 5
— positional `<path>` for the path form, `--mql <expr>` for the MQL
form, mutually exclusive at the call site. The verb controller
dispatches.

| Verb | Args | Force | Notes |
|---|---|---|---|
| `clone` | `<template>`, `--mql`, `--into`, `--here` | (no force) | Destination precedence: `--into <dest>` > `--here` (avatar's environment) > inventory fallback (the giver). The slot for `template.environment` lookup sits between `--here` and the inventory fallback — see [templates.md](./templates.md) for the broader spawn design. Resolves template path cwd-relative. No force-bypass: clone is "willing something new into existence" — there's no per-target witness to bypass; permissions are the only gate (future). |
| `reload` | `<target>`, `--mql` | (no force) | Hot-reloads a path. No force-bypass: reload operates on modules / prototypes, not on a Stuff target — there's no per-target witness to bypass. Permissions handle "are you allowed to reload this path?" |
| `destruct` | `<target>` (object), `--mql` | `-f` → `StuffApi.forceDestruct` | Target is `type: object` so MQL resolves it; the path-atom extension (step 2) lets bare paths address templates with no live clones. |
| `eval` | `<expr>?`, `--on <expr>`, `--all` | (none) | Eval has no per-target veto — the eval'd code's own logic + `@CallSecurity` decorators on whatever Apis it touches are the boundary. |
| `teleport` | `<target>` (object), `<destination>?` (`to <dest>`) | `-f` → `forceMove` fallback | Tries `Mobile.teleport` first; on Mobile-level veto, falls back to `ContainmentApi.move` / `forceMove`. |

## Force-bypass shape (recap)

Each `forceX` decorated method on a lowest-layer Api carries
`@CallSecurity(SecurityPolicies.AdminOnly)`. v1 the policy is an
always-deny stub: every `-f` invocation throws "admin privilege
required" before the body runs. The seam is in place; the real
permissions-aware policy replaces the stub when permissions land,
and no decorated method needs to change at swap time.

See call-security.md § AdminOnly and the force-bypass shape for the
parallel-API convention.

## Settings keyspace (`eval.*`)

Session-lifetime — declared in schema, but the runtime defaults are
baked into `EvalScript.run` until the sandbox honors per-call knobs.

| Key | Type | Default | Purpose |
|---|---|---|---|
| `eval.timeoutMs` | number | `1000` | Wall-clock budget per `eval` invocation. |
| `eval.maxDepth` | number | `32` | Maximum recursion depth inside an eval body. |

## EvalScript — Stuff-wrapped sandboxed eval

`mud/lib/script/EvalScript.ts`. The eval'd code lives inside a real
Stuff so it has world identity (MQL-addressable), an attachable
security frame (template-path derived, future-permissions-friendly),
and a clean stepping stone toward authored scripts.

Surface:

```ts
class EvalScript extends Idea {
  setCode(code: string): void;
  getCode(): string;
  async run(receiver: Stuff): Promise<unknown>;
}
```

Sandbox: Node's built-in `vm.Script` + `vm.createContext`. The
context exposes a curated allowlist (currently
`StuffApi`, `MqlApi`, `ContainmentApi`, `MixinApi`, `console`, plus
`self` / `target` aliases for the receiver). `process`, `require`,
`globalThis`, `fs`, `child_process` etc. are not reachable.

The sandbox is **not** the security boundary — `@CallSecurity`
decorators on the Apis are. The sandbox is the moat that prevents
trivially escaping below the api layer.

### Singleton placement

`/home/<playerId>/_eval` (with a fallback to
`/home/<stuffId>/_eval` for non-Avatar givers). Establishes the
`/home/` branch of the template tree as the per-player namespace;
future variants tag the basename (`_eval.<tag>`) instead of
nesting deeper. Each new `eval <code>` destructs the previous
singleton and clones a fresh one, then `setCode` + `run`. The
singleton is not persisted across server restarts.

`eval` (no expr) reuses the existing singleton — re-runs its
most-recent code against the supplied `--on` (or avatar by default).
This is what makes the singleton actually useful as an object: it's
the "clipboard" for the most recent eval body.

### Future expansion

Three additive features extend this pattern. All deferred from the
current slate; the v1 shape doesn't preclude them.

1. `--save <name>` — promotes the eval'd code to a persistent named
   template at `/tpl/script/<avatarId>/<name>`.
2. `--mixin <Name>` (repeatable) — composes additional mixins onto
   the eval'd Stuff. Allowlisted via a `composableViaEval: true`
   marker on the mixin's class; default-deny for unfamiliar mixins.
3. `--extends <ClassPath>` — extends a user-authored class. Depends
   on `--save` (the class must exist as a template first). Same
   allowlist guardrail as `--mixin`.

Locked permission rule for `--save`: a script's effective authority
= `min(template-path-derived authority, invoking caller's
authority)`. Eval can never grant more than the caller already has.

### Migration target

`isolated-vm` for real V8-isolate sandboxing. The public surface
(`setCode`, async `run`) is designed to stay stable across the
swap; only `EvalScript` internals change. Two accommodations baked
in v1: `run` is `Promise<unknown>` (so async-first callers don't
need a re-touch), and the sandbox-context shape is a definition-
local list of allowed names rather than a literal object scattered
through the code.

## See also

- [shell-workspace.md](./shell-workspace.md) — sister mixin
- [perceiver.md](./perceiver.md) — sister mixin (the `scry` verb)
- [lifecycle.md](./lifecycle.md) — `canDestruct` / `onDestruct` /
  `forceDestruct`
- [call-security.md](./call-security.md) — `AdminOnly` policy stub
  and the force-bypass shape
