# Git workflow (in-runtime VCS)

The **in-runtime VCS** subsystem turns runtime source edits into version
control. Authors edit engine TypeScript live through the CMS or the
`write` verb, and hot-reload makes the change take effect immediately —
but nothing captured those edits into git. `GitApi` closes that: a gated
surface that commits the live working tree and pushes it to GitLab, giving
**version history, review, rollback, and durability across redeploy** —
without opening any new authorization surface.

The problem it solves: before this, a runtime edit lived only on the box's
disk until the box was re-provisioned. There was **no git in the repo at
all** (no `simple-git`, no `child_process`), so an author's live work was
undurable, unreviewable, and unrecoverable. This subsystem makes a runtime
edit a first-class commit.

Seeded by the git-workflow slate + requirements + plan; the requirements
and plan retired at merge (this doc is the live reference; see *History*).
It is the *in-runtime VCS* brick the
provenance-slate reserved.

## The governing constraint: the working tree **is** the live server

The dev box runs `tsx` directly from its checkout, so that working tree is
the running engine's live source. One consequence dictates the whole
design: **you cannot use `git checkout <branch>` as an isolation
mechanism** — a branch switch would swap *every* author's running engine,
including files nobody touched. `git worktree` doesn't help either (a
second worktree is files the running process never executes).

So the model is **snapshot-and-push, not branch-and-isolate**:

- The box tracks **one long-lived `authoring` branch**.
- Runtime edits accumulate as working-tree changes.
- `publish` commits and pushes them (HEAD advances *along* the tracked
  branch — a commit changes no file on disk, so the running process never
  notices).
- **Isolation and review move up to the MR layer** (`authoring → master`,
  whole-branch this cycle). `master` is the reviewed line prod images are
  cut from.

`GitApi` is **topology-agnostic** — `publish` pushes `status().current`
(whatever branch the box tracks), so it works on `master` today and
`authoring` post-migration with zero code change. The box-on-`authoring`
migration is a separate, documented ops step (see
deployment.md § *In-runtime VCS*), deliberately **not**
in the feature merge.

## Shape at a glance

```
SURFACES                       ONE GATED CORE            STORE
git verb (shell) ───┐
                    ├─► GitApi ──► GitLogic ──► simple-git ──► the checkout
CMS git panel ──────┘  (facade)   (singleton)              ──► push → GitLab
(GitRoutes REST)
```

`GitApi` (`mud/api/git.ts`) is the gated forwarding shell over the
hot-reloadable `GitLogic` singleton at `/platform/idea/api/git` (reached via
`StuffApi.singletonSync`); it homes the one error class, `GitError`. Five
ops: `status` / `diff` / `log` (reads) and `publish` / `revert` (writes).
Neither surface reimplements the mechanism or the gate.

## The three path-spaces

`GitLogic` translates between three ways of naming a file — the load-bearing
plumbing detail:

| Space | Shape | Who speaks it |
|---|---|---|
| repo-relative | `packages/server/src/mud/platform/Foo.ts` | `git status`/`diff` porcelain |
| source-logical | `/server/src/mud/platform/Foo.ts` (packages-stripped, leading `/`) | `AccessApi.resolveSourceFolderZone` (walks the template tree) |
| absolute fs | `<repoRoot>/packages/…` | `simple-git` cwd |

The repo root (where `.git` lives) is `path.dirname(SourceTreeApi.getSandboxRoot())`
(the sandbox root is the `packages/` dir itself), memoised process-wide. A
path **outside `packages/`** (`docs/`, `deploy/`, root config) has *no*
source-logical form → the write gate resolves no title for it
(`ownerOf` → `null`, which fails closed) → **denied for a normal wizard**. That is intentional (git is never
a bypass), and it means a wizard editing `docs/` through the source tree
**cannot `publish` it** — a real, documented constraint.

## The security spine — permission-scoped operations

The governing invariant: **git can never touch a file a direct source
write couldn't.** Every operation resolves to an *affected-path set*, and
each path must pass the **same gate `_writeSource` uses** —
`AccessApi.isWizard(actor)` **and** `AccessApi.can(actor, 'write',
resolveSourceFolderZone(path))`. That gate is mirrored **verbatim** as a
module-private `gateSourceWrite` in `GitLogic` (a deliberate copy of
`CmsLogic.gateSourceWrite` / `WriteController._gateSourceWrite`, not a
shared extraction — the lowest-blast-radius choice; a shared
`AccessApi.canWriteSource` is a noted follow-up).

The acting principal is **derived from the execution context**
(`ExecutionContextApi.getActingAuthor`) — **never a caller-supplied
argument** (the CMS/provenance anti-spoof property); a `null` actor fails
every gate closed.

| Op | Affected-path set | Rule |
|---|---|---|
| `status` / `diff` / `log` | (read) | `isWizard`, no path gate |
| **`publish`** | the **dirty set** (staged ∪ unstaged ∪ untracked) | stage **only** the writable paths; commit those; leave the rest dirty for their owner → **per-owner slicing falls out for free** |
| **`revert <sha>`** | the paths in that commit's diff (`diff-tree`) | require **all** writable; **any** out-of-scope path **rejects** the whole revert (no partial revert) |

Non-path-scopable history rewrites (`reset` / force-push) are **out of
scope** this cycle — rollback is additive `revert` only.

`publish` handles a push rejection (protected branch / non-fast-forward)
as `pushed: false` with the commit **retained locally** (durable,
re-pushable), never an auto-reset. `revert` on a conflict aborts (`revert
--abort`) and surfaces `GitError('conflict')`.

## The identity model — one credential, per-avatar authorship

Git separates *pusher* / *committer* / *author*, so only the push
credential is shared:

- **Pusher** — one shared machine token, `GITLAB_PUSH_TOKEN`, read **only**
  inside `GitLogic`, injected per-command via `-c http.extraHeader` bearer
  (never a remote URL, so it never reaches `git remote -v`). A `sanitize()`
  scrubs it from any surfaced git output, returned field, or error string.
- **Author** — every commit is `--author`'d to the acting avatar,
  `<Name> <playerId@saxonberg.local>` (a **synthetic** email — the repo is
  public; real Google addresses must not enter history). Hard-derived from
  the context actor, never caller-supplied. The git author is a
  convenience **mirror** of the authoritative `AuthoringEvent` provenance
  ledger.
- **Committer** — a fixed machine identity (`Saxonberg <box@saxonberg.local>`,
  set per-command) so commits succeed on a fresh checkout with no git
  identity, without disturbing the per-avatar author.

## The two surfaces

Both are thin over the one gated `GitApi`; both derive the actor from
context and hold no authz beyond their entry gate.

- **The `git` verb** — `content/platform/cmd/system/git.yaml` + `GitController`
  (category `system`, AuthorMixin-afforded, gated by the YAML
  `requiresWizard` validator). Subcommands `status` (default) / `diff` /
  `log` / `publish -m <msg>` / `revert <sha>`; reads render through
  `MessageApi.scene`, failures surface as `controller-rejected` notes. The
  verb is afforded to every operator surface but a non-wizard can't run it.
- **The CMS git panel** — `GitRoutes` (a **sibling** of `CmsRoutes`; the
  git ops don't fit the `backend/path/body` shape) exposes
  `GET /api/git/{status,diff,log,csrf}` + CSRF-protected
  `POST /api/git/{publish,revert}`, each binding 1:1 through the
  `CmsSession.runAsSessionPlayer` attribution bridge — **no authz in the
  route layer**, writes reuse the shared CMS double-submit CSRF token. The
  client `CmsGitPanel` is a fourth tab in `CmsSurface` (REST-poll, no WS),
  a dumb server-authoritative renderer over `gitClient`.

## Redeploy durability

`publish` pushes to GitLab immediately, so a redeploy reconstitutes every
published edit: `deploy/dev/update.sh`'s `git pull --ff-only` runs on the
box's checked-out branch (its `--ff-only` was chosen to anticipate `GitApi`
— it refuses to clobber unpushed local work rather than lose it). After a
`publish`, local and remote `authoring` are in sync, so the pull is a clean
fast-forward. The migration details (box checkout, the `deploy-dev`
CI-rule change, `GITLAB_PUSH_TOKEN` provisioning) live in
deployment.md.

## Deferrals

- **Content / document → git** (templates + scripts live in Mongo, not
  files) — needs a Mongo→file **export** bridge, deferred to the
  content-packs round-trip.
  Source plane only.
- **History rewrites** (`reset` / force-push) and their admin-tier gating.
- **Finer-than-branch review** (slice / commit / author-scoped MRs) — rides
  the CMS law==code forums-review gate; the permission-floor `publish`
  already produces per-owner commits as its substrate.
- **Per-user `/home/<id>/` submodules** — blocked on home code being
  file-backed (today it's Mongo `documents`).
- **Auto-open-MR** (GitLab API), commit signing, in-CMS merge-conflict
  resolution, a shared `AccessApi.canWriteSource` extraction.

## History

Built as `feature/git-workflow` (the `feat(git):` commit range on the
branch off `origin/master`). Aligned with the plan; the plan's leaned forks
were taken as-is at build time: D1 = verbatim gate copy (not a shared
static); `GITLAB_PUSH_TOKEN` the env var; `publish` scoped by permission
(not additionally by author); `simple-git` the dependency; revert-only
rollback. One build-time judgment call worth recording: **every commit
carries a fixed machine *committer* identity** (distinct from the
per-avatar `--author`) so commits succeed on a checkout with no configured
git identity — the author assertion (`%an/%ae`) is unaffected. The
real-remote authenticated push is the one path tests can't cover (they use
a local bare remote); its token-secrecy invariants are enforced in code.

## ⭐ The git panel is a CARD

Bare `git` (which falls through to `status`) opens the `git` card as
well as printing. Its body is the client's own `/api/git/*` surface —
a `client`-source card, so the server owns its existence, identity,
lifetime and pinned-ness and the client owns only the body.

⚠ **Only the bare / status form.** `git publish` is an ACT, and an act
that also rearranged your workspace would be doing two things on one
keystroke.

⚠ Pinned by default and `noProse`: it has no terminal rendering to
degrade to, so a `terminal`-only `shell.result` must leave it alone.

See [card-surface.md](./card-surface.md).
