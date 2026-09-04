# Access

The access subsystem is the permission seam that `call-security`
explicitly reserved: a thin **`AccessApi`** facade with six
predicates plus a path-resolver helper, plus a **narrow-entry
pattern** for privileged mutations. The substrate lands on top of
the existing `grouping` (membership) and `zone` (inheritance walk)
substrates.

State and behavior live on the singleton **`AccessRegistry`**
Stuff at `/platform/idea/AccessRegistry`; `AccessApi` is a thin facade that
delegates through the security gate. Every public Registry method
carries `@CallSecurity(FromModule('/api/access#AccessApi'))`, so
the only legitimate calling path is through the Api — external code
that grabs the Registry instance via `StuffApi.findByTemplatePath`
gets a reference but `SecurityError` thrown on any method call.

## ⚠⚠ Every predicate fails CLOSED when the Registry is absent

`lookupRegistry()` returns null before `/platform/idea/AccessRegistry` is
registered — during early boot, and in any test that does not stand one
up. Five predicates (`can`, `canMutateZone`, `isWizard`, `isStreamer`,
`isArchwizard`) used to answer **`true`** in that window, undocumented.

**A missing authority is not a grant.** `isWizard` is the code-trust
axis — it gates `eval`, `reload`, source-tree writes and the executable
code-naming fields on a content template — so an unanswerable question
resolving to *yes* made the most dangerous door in the system the one
that opened when the lock was missing.

⚠ The cost is real, and it is why it was written the other way: a test
that exercises a gated path must now stand up a Registry or stub the
predicate. Closing it turned up **12 tests across three files** that
were passing only because the gate was absent — clone placement,
dorm provisioning and the release write transport. None was testing
authorization; each now declares the permission answer it depends on, at
its own setup, where a reader can see it. *A test that passed only
because the gate was missing was not testing the gate.*


## The two account axes, and title

Capability on this platform is **title over a resource, within your
extent** — never a tier. There is no author tier (content-packs wave 3
deleted `AccessApi.isAuthor`, the `requiresAuthor` validator and the MQL
permission snapshot): an *author* is anybody, and what they may do is
answered per resource by the title that covers it. Beside title stand
two **account axes** that are not about content at all — code trust
(`wizards`, conferred by `archwizards`) and the livestream control
plane (`streamers`).

The substrate ships these predicates:

1. **`AccessApi.can(subject, action, resource)`** — resource-
   targeted ownership check. Resolves the resource's zone
   templatePath, then `ParcelApi.ownerOf(path)` (title → self-home,
   longest-prefix over parcel extents — see [parcel.md](./parcel.md);
   `null` when nothing covers the path), and dispatches on the owner
   kind: a **group** owner resolves to a ref and checks
   `GroupApi.isMember`; a **player** owner is an identity match; an
   **organization** owner admits its staff (`EmploymentApi.holdsPosition`)
   and its appointing authority (`EmploymentApi.holdsAuthority` — the
   office holder, founder default included). **A `null` owner denies** —
   untitled is nobody's. A non-resident organization also denies, with
   one `access.organization-owner` diagnostic per path (the organization
   must be a `boot:` entry of the pack that claims for it). The action is
   a free string; this build doesn't filter ownership by action.
2. **`AccessApi.canMutateZone(subject, zone)`** — role-gated. For a
   **group** owner, true iff `subject` has the `'owner'` role in the
   parcel's governing group; for a **player** owner, an identity
   match; for an **organization** owner, the same staff-or-authority
   test. Used by verb controllers when the target IS a Zone
   Template (transfer ownership, destruct the slice) — and by the
   `subdivide` / `transfer` parcel verbs. Resolves title the same
   way (`ParcelApi.ownerOf`); the covering parcel's owner is the
   nearest-ancestor owner the former upward walk found.
3. **`AccessApi.heldExtents(subject)`** — the inverse read: every
   `parcels` row whose holder admits the subject through the dispatch
   above, plus the subject's own `/home/<key>` root. This is the
   **within-your-extent pattern**: a verb that used to ask "is this
   person staff?" now asks "where does this person hold?" and scopes
   itself — `broadcast --at <extent>` reaches the extent you name and
   hold, `teleport` moves you between two points inside ONE held
   extent, `find` lists paths under your extents, the CMS tree prunes
   to them, `errors` shows the diagnostics under them. A non-holder is
   told what they *do* hold, never "you are not an author."

   ⭐⭐ **The right frame is AUTHORIAL AUTHORITY, not privilege** (TPA
   reform D11). Free movement inside an extent you hold is not a perk
   the engine grants the powerful; it is the observation that *moving
   around inside what you author is not a journey*. The same authority
   that lets you edit the place lets you be in it. Cross the boundary
   and it is the Teleport Authority like everyone else, and the wizard
   axis — code trust — buys no movement at all.

   ⚠ **A use-grant is not authorship, and it is excluded structurally.**
   `heldExtents` admits on `ParcelRecord.getOwner()` and never reads
   `grants[]`, so somebody who *leases* a place gets none of this and
   nothing had to be written to exclude them. That is worth stating
   because the tempting fix — "let a grant-holder move around what they
   lease" — quietly makes a lease into title, and the whole point of the
   grant vocabulary is that it is *not*.
4. **`AccessApi.isWizard(subject)`** — orthogonal wizard axis, the
   **code-trust capability**. True iff `subject` is in `'wizards'`.
   Determines who can write TypeScript source, run `eval`, `reload`
   modules, AND set the executable code-naming fields
   (`class` / `hydratorClass` / `behaviors[].brain`) on a content
   template (see *The code-trust lockdown* below). Doesn't matter
   what slices you own; the question is whether you have escape
   capability. A content author who is *not* a wizard is a
   **protowizard** (content-write access without code trust — an
   unstored complement, not a group).
5. **`AccessApi.isStreamer(subject)`** — orthogonal streamer axis.
   True iff `subject` is in `'streamers'`. Gates the livestream
   control plane (the `stream` verb; later scene / lower-third /
   afk). Distinct from the wizard axis — a streamer drives the
   broadcast overlay without holding TS-escape capability. See
   [livestream.md](./livestream.md).
6. **`AccessApi.isArchwizard(subject)`** — orthogonal archwizard
   axis, the **wizard-conferral capability**. True iff `subject` is
   in `'archwizards'`. Archwizards confer/revoke wizard status via
   the `wizard grant/revoke <player>` verb (the `requiresArchwizard`
   validator). Archwizard membership itself is operator/root-managed
   (env seed + the `group` verb); the chain is
   `operator/root → archwizards → wizards`, with a Prime Minister
   office deferred above `archwizards`.

Plus one helper for slice-aware workspace verbs:

- **`AccessApi.resolveSourceFolderZone(sourcePath)`** — walks a
  source path against the template tree most-specific-first,
  returning the closest extant FolderZone instance. Workspace
  controllers in source/mirror mode pass the resolved zone as the
  access resource.

## `AccessRegistry` Stuff

The Registry is an `Idea + PostRegistrationMixin` singleton at
`/platform/idea/AccessRegistry`. Instance state:

- `cachedWizardsRef` / `cachedStreamersRef` / `cachedArchwizardsRef` —
  resolved `GroupRef`s for the three bootstrap-seeded axis groups.
- `cachedWizardPlayerIds` / `cachedStreamerPlayerIds` /
  `cachedArchwizardPlayerIds` — Sets of the **member keys** in `'wizards'` /
  `'streamers'` / `'archwizards'`; warmed lazily on first
  `isWizard` / `isStreamer` / `isArchwizard` call, invalidated via
  the managed provider's `onChange` callback.

**Group membership keys on the member's `templatePath`, uniformly** — a player
as `/platform/agent/Avatar/<playerId>`, an NPC (a staff agent) as its own path. There is
NO player-vs-NPC branch anywhere: the member key of any subject is just
`subject.getTemplatePath()` (`memberKeyOf`), and an authority roster (which
holds only players) simply never contains an NPC's path. The bare `playerId`
stays the **auth/account** identity (`getPlayerId`, `User.playerIds`, session,
the env `WIZARD_PLAYER_IDS`-style seeds and the `wizard grant <playerId>` verb)
— those boundaries convert once via `Avatar.getTemplatePath(playerId)`. Offices
(single-holder seats) stay playerId-keyed — they are player-only and never
exhibit the avatar-vs-NPC mix.
- `reportedUnresolvedOrganizations` — the paths already warned about
  (one `access.organization-owner` diagnostic per non-resident
  organization holder, not a flood).
- `wizardCacheCancel` / `streamerCacheCancel` /
  `archwizardCacheCancel` — onChange cancellation handles, cleared
  on destruct.

`postRegister` runs idempotent bootstrap seeding of the **tag-like** groups
(whose membership comes from env vars, not zone ownership). There is
**no `core` group** (content-packs wave 3): title is the packs'
manifests' to declare, and the platform's own roots are held by the
executive.

1. Mint `'wizards'` Group if absent (no FolderZone stamp — it's a
   tag-like group whose only role is gating the `isWizard`
   code-trust axis), then add any playerIds from the
   `WIZARD_PLAYER_IDS` env var (additive + idempotent).
2. Mint `'streamers'` Group if absent (no FolderZone stamp —
   tag-like, gates the `isStreamer` axis), then add any playerIds
   from the `STREAMER_PLAYER_IDS` env var (comma-separated,
   additive + idempotent — never removes).
3. Mint `'archwizards'` Group if absent (no FolderZone stamp —
   tag-like, gates the `isArchwizard` conferral axis), then add any
   playerIds from the `ARCHWIZARD_PLAYER_IDS` env var (additive +
   idempotent).

**Zone-ownership is NOT resolved here anymore.** As of property phase 0a,
ownership moved out of the editable `domain` zone template into the gated
`parcels` collection (the governing security invariant). The `ParcelRegistry`
owns the title store + the mint-or-find group-ref resolution; `postRegister`
seeds only the tag-like groups above. The former data-driven
`effectiveOwnerRef` / `resolveOwnerGroupName` machinery is retired.

Re-running boot against a populated DB is a no-op (existing
Groups are not overwritten; member seeding only adds missing ids).

## Ownership: the parcel layer

Ownership is no longer stamped on `Zone` — the three ownership fields
(`ownerGroup` / `accessGroups` / `ownerGroupName`) were **removed** in
property phase 0a. Title is a **parcel** in the gated `parcels` registry,
resolved by longest-prefix over parcel extents. `AccessApi.can` /
`canMutateZone` / `canAtPath` / `heldExtents` consult `ParcelApi` for
ownership; the access layer keeps only the *decision* logic (group
membership/role vs. player identity vs. organization staff/authority).

**`canAtPath(subject, action: TreeAction, path)`** (content-packs wave 2)
is the **path-targeted** form for the path-addressed trees: the
covering owner via `ParcelApi.ownerOf(path)` — rung 1 a parcel, rung 2
the self-home, otherwise `null` and a denial — then the same `can()`
dispatch of that owner. No zone step, no state default. `TreeAction` is
the closed vocabulary `'write-document' | 'write-template' |
'write-source' | 'install' | 'read' | 'broadcast' | 'teleport'`:
`write-document` is the document store's gate
([document-store.md](./document-store.md)) and the soul committee's
(`soul` writes under `/expression`); `write-template` the template
store's; `install` is `pack` (`requiresPackInstaller` — holding
`/compact/executive`); `read` the CMS per-path read
([cms.md](./cms.md)) and the pack-maintainer diagnostics read;
`broadcast` the `broadcast --at` gate; `teleport` names the hop.
See **[parcel.md](./parcel.md)** for the registry, `grant`, the
`ownerOf` chain, the sparse hierarchy, and the `subdivide` / `transfer`
verbs.

The `accessGroups` flat-union multi-group ACL is a **conscious deletion to
be re-provided** — no seed used it (byte-identical removal), and the
multi-group / secondary-access capability returns as 0b's `grants[]` seam.

## The narrow-entry pattern

A new engine substrate introduced by this build. A privileged
mutation Api method gets a `FromController(...)` policy
restricting it to one (or a few) verb controllers, and those
controllers do the access check via `AccessApi.can` before
invoking. Combined, the mutation has exactly one legitimate
entry path AND that path enforces who is authorized.

Adoption sites:

- `AccessApi.setWizardMembership(playerId, makeWizard)` → gated by
  `FromModule('/platform/idea/cmd/author/WizardController')`
  (string-keyed `FromController(WizardController)`). `WizardController`
  (the `wizard grant/revoke` verb) is the sole legitimate caller; its
  `requiresArchwizard` validator enforces *who* may invoke (the giver
  must be an archwizard). The mutation resolves the `wizards` group,
  add/removeMember, save, and `fireChange` to invalidate the wizard
  cache. The wizard-conferral act has its own auditable entry path.
- `StuffApi.forceDestruct` → gated by
  `FromModule('/platform/idea/cmd/author/DestructController#DestructController')`
  (the string form of `FromController(DestructController)` —
  string-keyed to avoid a value-level static-import cycle).
  `DestructController` runs `AccessApi.can(giver,
  'force-destruct', target)` on the force branch (and
  `'destruct'` on the polished branch). Zone targets route to
  `canMutateZone` instead.
- `ContainmentApi.forceMove` → gated by
  `AnyOf(FromModule(TeleportController), FromModule(GotoController))`.
  Each controller's `execute` runs `AccessApi.can(giver,
  'force-teleport' | 'force-goto', ...)` on the force branch
  (and `'teleport'` / `'goto'` on the polished branch).

The Registry itself is the access-side instance of the same
pattern: every public Registry method carries
`@CallSecurity(FromModule('/api/access#AccessApi'))`, so
external code's only reachable surface is the Api facade.

## The code-trust lockdown

In a Node + proxy-security stack, anyone who can author a line of
TypeScript can subvert the whole security apparatus — so that power
is the **wizard** capability and nothing else. The catch: a content
template is *data*, but several of its fields resolve to executable
code, so "can write content" silently grants "can run code" unless
those fields are gated. This build closes that bypass.

**The wizard / protowizard partition.** A **protowizard** is the
unstored complement of a wizard: any actor with content-write access
(`can(…, 'write', …)`) who is **not** in `'wizards'`. There is no
`protowizard` group or flag — "can edit content, can't write code"
falls out of the existing content gate plus the code-field gate below.

**The direct gated set.** Three template fields name a module export
directly and are **wizard-only-writable**:

- `class`
- `hydratorClass`
- `behaviors[].brain`

(`CodeNamingFields.FIELDS` is the single source of truth.) The gate
lives at the **universal `TemplateApi.saveTemplate` chokepoint**
(`TemplateLogic.enforceCodeFieldGate`), through which both the
in-world `write -c` verb and the REST CMS funnel. The actor is derived
from the execution context (`ExecutionContextApi.getActingAuthor` —
never caller-supplied; the *gated-api-actor-from-context* rule). The
allow ladder:

1. no attributable Avatar author (system / bootstrap / forced /
   cross-actor / pre-Avatar login + char-gen + guest provisioning) →
   **ALLOW** (the provisioning invariant);
2. a wizard → **ALLOW**;
3. else (a protowizard) → the **delta rule**: reject any write that
   *introduces or changes* a direct field — `class` / `hydratorClass`
   inequality vs. the existing doc, or an incoming `behaviors[].brain`
   multiset that is not a subset of the existing one. A pure cosmetic
   edit (same class/hydrator, brain set unchanged-or-reduced) passes —
   the protowizard authoring path.

**The transitive set closes by construction.** The reference fields
(`adornments[].template`, `exits[].destination`, `exits[].door`,
`props[]`, `cast[]`, `container`, `warren`, `startLocation`,
`routes[].to`/`.warren`) get **no per-field gate**: each names *another
template*, which must itself have passed the `class` gate, so a
reference can only ever instantiate a wizard-vetted-or-protowizard-safe
class. (Proven in `TemplateLogic.transitiveClosure.test.ts`.)

**The structural carve-out (D4).** A `mkdir`-shaped write — a
Zone/folder `class` with no behaviors and the standard (or absent)
hydrator — is exempt: a folder class is engine code by construction,
carries no author-chosen executable strategy, and is constrained by
the folder/leaf invariant. So a protowizard can still create
organizational sub-zones.

**The `cp`/`mv` tightening (D5).** `cp`/`mv` content copy an existing
template's `class`/`hydrator`/`behaviors` into a *new* path (no
existing doc at dst), so the delta rule makes them **wizard-only** for
class-bearing leaves. Protowizards author by cloning/customizing
wizard-made templates and editing cosmetic `data`; they never type a
class or brain freehand. (The v2 relaxation — letting protowizards
*pick among* a wizard-vetted catalog of safe classes/brains — is
deferred.)

**The one bypass, independently gated (D6).** Every *authoring*
template write funnels through `saveTemplate` except `PackLogic`
(content-pack import, which writes `PersistApi.save` directly). The
`pack` verb is `requiresWizard`-gated, so a non-wizard can never reach
that path. Avatar persist-back no longer writes the `domain` collection at
all — it captures runtime state into the separate `holder_snapshots` store
via the self-persistence spine, which drift-guards fields to declared
`fieldMeta`'s persistent entries (never an author-named `class`/`hydratorClass`/`brain`
string) and reconstitutes items only through the gated `StuffApi.clone` — so
it cannot forge a code-naming field. See
[persistence.md § The self-persistence spine](./persistence.md#the-self-persistence-spine-persistable).

**The drift-guard.** `codeNamingDriftGuard.test.ts` enumerates every
module-resolving call site under `mud/` (`resolveExport` /
`resolveExportSync` / `loadClassByPath` + non-literal dynamic
`import()`) and asserts the set equals a classified manifest, so a new
field that resolves a module export can't silently escape the gate.

## Where the gate lives — validator vs controller body

The dispatcher's validator phase is the right home for an access check
when the decision is sync-decidable from the giver alone (no need for
the resolved model). The declarative validators cover the simpler
cases:

- **`requiresPackInstaller`** — `canAtPath(giver, 'install',
  '/compact/executive')`: holding the executive (the PM and her staff)
  is what installs packs. Used by `pack`. (The former
  `requiresCoreAccess` — "member of the state group" — is gone with
  the group: `soul` is now title over `/expression`, held by the `soul`
  group the platform declares; `broadcast` is `broadcast --at <extent>`,
  title over the extent named — both checked in the controller body
  because the extent is part of the model.)
- **`requiresWizard`** — `isWizard(giver)`. The code-trust gate;
  used by `eval`, `reload`, `config`, `practice`, and the
  banking operator verbs.
- **`requiresStreamer`** — `isStreamer(giver)`. Used by `stream`.
- **`requiresArchwizard`** — `isArchwizard(giver)`. Used by
  `wizard grant/revoke`.

All follow the typed-preload pattern documented on
`CommandValidator<T>`: the async preload returns the boolean
decision (`AccessApi.can(...)` / `isWizard(...)` / `isStreamer(...)`); the
dispatcher captures it in a per-dispatch `ValidatorPreloads` map and
passes it back to the sync validator body as its second argument
(`preloaded`). No module-level state, no manual cleanup.

The model-dependent cases — `destruct` / `teleport` / `goto` (force
flag + target stuff + Zone-target detection), `clone` (source
template), and the workspace verbs (tree-mode branching, dual-
endpoint cp/mv, source-path resolution) — stay in the controller
body. CommandValidator's `preload(context)` signature doesn't expose
the resolved model, and splitting the branching across many narrow
validators trades one controller check for several preload+sync
pairs. When the matrix simplifies (an inert force flag, a single
tree mode, etc.) we revisit.

## Verb-controller gates (matrix)

| Controller | Check |
|---|---|
| `DestructController` | Zone target: `canMutateZone(giver, target)`. Else non-force: `can(giver, 'destruct', target)`; force: `can(giver, 'force-destruct', target)`. |
| `TeleportController` (the **tpa pack**'s) | `heldExtents(giver)` — from and to both inside ONE held extent, which makes the hop free (D11). No other-subject arm: relocating an object moved to `goto --subject` (TPA reform P13). |
| `GotoController` | non-force: `can(giver, 'goto', dest)`; force: `can(giver, 'force-goto', dest)`. With `--subject`: `can(giver, 'teleport'`/`'force-teleport', subject)` — the authority is over the OBJECT, not the destination. |
| `SoulController` | `canAtPath(giver, 'write-document', '/expression/emotes/<verb>')` — the soul committee's title ([emotes.md](./emotes.md)). |
| `BroadcastController` | `canAtPath(giver, 'broadcast', extent)` for `--at <extent>`; a non-holder's refusal lists `heldExtents(giver)`. |
| `PackController` | `requiresPackInstaller` (validator) — `canAtPath(giver, 'install', '/compact/executive')`. |
| `EvalController` | `requiresWizard` (validator) — `isWizard(giver)`; no slice (eval is TS execution). |
| `StreamController` | `requiresStreamer` (validator) — `isStreamer(giver)`; no slice (livestream control plane). |
| `WizardController` | `requiresArchwizard` (validator) — `isArchwizard(giver)`; calls the narrow-entry `setWizardMembership` (wizard grant/revoke). |
| `CloneController` | `can(giver, 'clone', sourceResource)` — slice walk on source path. |
| `ReloadController` | `requiresWizard` (validator) — `isWizard(giver)`; no slice. |
| `WriteController` content | Zone target: `canMutateZone(giver, target)`. Else: `can(giver, 'write', target)`. |
| `WriteController` source/mirror | `isWizard(giver)` AND `can(giver, 'write', resolveSourceFolderZone(path))`. |
| `MkdirController` content | `can(giver, 'mkdir', parent)` flat — sub-zone creation is a member-level op. |
| `MkdirController` source | wizard + slice. |
| `RmController` content | Zone target: `canMutateZone(giver, target)`. Else: `can(giver, 'rm', target)`. |
| `RmController` source | wizard + slice. |
| `CpController` | Source endpoint: READ rules (slice walk only). Dest endpoint: WRITE rules per tree mode. |
| `MvController` | Both endpoints WRITE rules — `mv` REMOVES source after write. |
| `LsController` / `CatController` / `GrepController` | source/mirror mode: `can(giver, 'read', resolveSourceFolderZone(path))`. Content-tree reads are public. |
| `PwdController` / `CdController` | **no access check** in any mode (state queries on the actor, not file reads). |

## Subject = current command giver

The subject of every `can()` check is the current `CommandGiver`
resolved via `ExecutionContextApi.getCurrentCommandGiver()`. NPCs
nested under a staff player's command chain are NOT in the staff
group and therefore do NOT inherit the player's authority — an
invariant, not a leak.

`playerIdOf(subject)` extracts an Avatar's `playerId` (NPCs return
null and fail closed). `getCurrentCommandGiver()` returning `null`
(no command in flight) evaluates against an unauthenticated subject
and fails closed.

## Source-path → template-path resolution

`resolveSourceFolderZone(sourcePath)` walks the source path
against the template tree most-specific-first:

- `domain/lounge/Bar.ts` → tries `/world/lounge/Bar` (no match) →
  walks up to `/world/lounge` (match, extant FolderZone) → returns
  it.
- `lib/security/SecurityPolicies.ts` → walks up → no FolderZone
  match → returns `null` (the caller's title check then fails closed).

The lounge holds **two** titles — `/stuff/idea/lounge` (the template
namespace; `/lib/lounge` before the taxonomy refactor) and
`/world/lounge` (the content). Only the `domain/` one has a matching
*source* directory, so only that one is reachable by this walk. The
namespace title is live and load-bearing as a `parcels` row; its
source-tree half has always been vestigial, and no
`src/mud/lib/lounge/` directory ever existed.

Workspace mutation controllers in source/mirror mode pass the
resolved zone as the access resource; source paths inherit the
filesystem-style convention (longer paths shadow shorter ones;
the nearest extant FolderZone wins).

## MQL is not a permission

**Resolving a query is not a capability** (content-packs wave 3). The
former pre-resolution gate — an `isAuthor` snapshot the dispatcher
stamped on `ctx.permission`, `gateAuthor` throwing `MqlPermissionError`
on path seeds / `:world` / `mixin.X`-style filters, the `:admin`
predicate over a `coreMemberIds` set — is deleted with the author tier.
Any query resolves for anybody; what you may *do* to a result is the
verb's question, answered by title over that resource (`can`) and by
reachability (`flower:online` names a thing across the world; `attack`
still needs it in reach). See [mql.md](./mql.md).

## The bootstrap-seeded groups

`postRegister` mints three **tag-like** groups (membership from env vars, not
zone ownership):

| Group | Owner | Purpose |
|---|---|---|
| `'wizards'` | `'system'` | Code-trust (TS-escape) capability. Members can `eval`, `reload`, write source, and set the code-naming fields on a content template. Env-seeded from `WIZARD_PLAYER_IDS`. |
| `'streamers'` | `'system'` | Livestream control plane. Members can run the `stream` verb. Seeded from `STREAMER_PLAYER_IDS`. See [livestream.md](./livestream.md). |
| `'archwizards'` | `'system'` | Wizard-conferral capability (the `wizard grant/revoke` verb). Seeded from `ARCHWIZARD_PLAYER_IDS`. |

All start empty (bar any env seeds) — with no members, every gated path
denies (the secure default). **Content-slice owner groups** (`'lounge'`,
`'terminus'`, `'soul'`, `'wiki-editors'`, …) are **not** seeded here:
they are declared by a pack manifest's `requires.groups` and
`ensureGroup`d by name at install (found if present, minted empty
otherwise). Adding a new owned area is a manifest claim (see
[parcel.md](./parcel.md), [content-packs.md](./content-packs.md));
adding a wizard/streamer/archwizard is a single member-add. The
test-auth seam's founder character (`backend/TestHooks.ts`) joins
**every managed group** — whatever the installed packs declared, never
a list kept in the kernel — and never a state group; its authority
over the platform comes from heading the executive.

## Action vocabulary

`can(subject, action, resource)` takes the action as a free
string. This build doesn't filter ownership by action — any
member-of-permitted-group authorizes any action. When a real
per-action need lands (a read-only-grant, an audit-only-grant),
the grant shape extends then.

The vocabulary in use today: `'destruct'` / `'force-destruct'` /
`'teleport'` / `'force-teleport'` / `'goto'` / `'force-goto'` /
`'soul'` / `'broadcast'` / `'clone'` / `'write'` / `'mkdir'` /
`'rm'` / `'read'`. `canAtPath` takes the **typed** `TreeAction` instead
(see *Ownership: the parcel layer*).

## HMR notes

- Reload of `api/access.ts` invalidates the cached pointer
  (`AccessApi._resetRegistryRefForReload()`); Registry state is
  unaffected.
- Reload of `obj/AccessRegistry.ts` re-clones the Stuff per
  HotReloadApi's pattern. State resets; `postRegister` re-runs
  idempotently; caches re-warm lazily on first read. The
  `wizardCacheCancel` / `streamerCacheCancel` /
  `archwizardCacheCancel` handles are
  cleared in `onDestruct` so the leaked subscriptions don't
  survive.
- `ManagedGroupProvider.findByName` is the by-name lookup used
  by both bootstrap seeding and the wizard-cache warm path.
- **A holder group deleted at runtime** is benign: the cached
  `GroupRef` points at a deleted Group and `GroupApi.isMember`
  against it returns `false`, so every path under that title denies
  until the next install's `ensureGroup` re-mints it (empty). A
  `GroupRegistry`-side change-notification subscription is a future
  tighten-up.

## What's NOT in this build

These are deferred either because no consumer needs them today,
or because they're their own conversation:

- **Audit sink wiring** (Pillar 5 of call-security). Denial
  paths emit `controller-rejected` envelope notes and scene
  messages; nothing more.
- **Possession capability source** (keys/badges).
- **Ownership-of-personal-stuff capability source.**
- **Location/context capability source.**
- **Action-level enforcement for non-staff verbs** (chat gag-
  as-deny, door locks, channel post permission, field-mask
  access).
- **Deny composition** ("deny-wins"). No deny source ships
  yet.
- **`getResponsibleAvatar` / `ByResponsibleAvatar` /
  `ByActingAvatar`** — retired from the call-security roadmap.
- **Spoiler SEE/KNOW read gating** — own slate.
- **Read gating beyond source/mirror workspace verbs.**
  Content-tree reads stay public.
- **Tier vocabulary** (`'player' | 'builder' | 'wizard' |
  'owner'` from the slate). Authority is group membership at
  the zone-walked owner.
- **Admin-override entries for other Document collections**
  (`forceDeleteGroup`, `forceDisbandChannel`, `forceEditEmote`).
- **Per-action filtering on grants.**
- **The v2 wizard-curated class/brain catalog.** v1 makes the
  direct code-naming fields *wizard-only-writable* (a protowizard
  cannot set them at all — they author by cloning/customizing
  wizard-made templates). The relaxation that lets protowizards
  *pick among* a wizard-vetted set of safe classes/brains is a
  later build. (The bare "class-allowlist gap" this section used to
  flag — a lounge member writing `class: /lib/eval/EvalScript` and
  cloning it — is **closed** by *The code-trust lockdown* above.)

## Nearest title decides — there is no override

A player in `'lounge'` can write content under `/world/lounge/`
(the covering parcel is the lounge's; member passes). The executive
holds `/domain`, but `/world/lounge` is the *nearer* title, so the PM
writing under it walks to the lounge group — and is refused unless a
member. Scoped titles **replace** the enclosing one for their slice;
they don't add to it. The former open question ("should the state
group override?") closed with the state group's deletion: there is no
operator principal to win, only the holder of the nearest title.

## The committee — the title-holding principal, named

The jargon standard's replacement for cabal/"content group", realized
(the civics build): **a committee is whoever holds parcel title over a
subdivision** — a relationship *derived* from `ownerOf`, never a stored
kind. `CommitteeView` is a discriminated union: `{ kind: 'group', name,
groupRef, subdivisionPath }` for a group-held title and `{ kind:
'organization', … }` for an organization-held one (the executive over
the platform's roots, a corpo over `/corpo/<key>`). A player-held
subdivision has none; an untitled path has none. Reads live on
**`CompactApi`** (the single meta-institution facade — `api/compact.ts`):
`committeeOf(path)`, `isCommitteeMember` (group membership, or the
organization's staff/authority, or the founder via the Art. XI
pool-of-one backstop — mirroring the office founder default),
`committeeMembersOf`, and the chat seam
(`committeeChannelOf` / idempotent `ensureCommitteeChannel` — a bound
channel whose audience IS the committee group; chat.md). Surfaced by
the public `committee` verb (`system` category). A committee's
*authority* is exactly title — this Api adds identity and reads, never
a second authorization path; `can` stays the gate.

## Cross-references

- [call-security.md](./call-security.md) — `FromController`
  narrow-entry pattern, async `allows` contract, the framework
  this builds on.
- [grouping.md](./grouping.md) — `GroupApi.isMember` / `roleOf`,
  `ManagedGroupProvider.findByName`.
- [parcel.md](./parcel.md) — the **real-property title** substrate
  ownership resolution moved to (property phase 0a): `ParcelApi.ownerOf`,
  the coverage index, the `subdivide` / `transfer` verbs, and the
  `Zone` ownership-field removal.
- [zone.md](./zone.md) — `Zone.lookupField` inheritance walk,
  `ZoneApi.getEnclosingZone` (the ownership fields it used to
  document now live in the parcel layer).
- [response-envelope.md](./response-envelope.md) —
  `controller-rejected` Note shape used on access denials.

## History

Property phase 0a (`feature/property-0a-title`, commit `81b250be`) moved
ownership out of the editable `domain` zone template into the gated
`parcels` collection — the governing security invariant (access-check data
unspoofable by content edits). `can` / `canMutateZone` (and the since-deleted
`ensureAuthorGroups`) were repointed onto `ParcelApi.ownerOf`; the three `Zone` ownership fields
and the data-driven `effectiveOwnerRef` / `resolveOwnerGroupName` machinery
were removed. The dispatch grew a **player-owner** case (identity match)
alongside the group case (membership/role), since parcels can be titled to an
individual. See [parcel.md](./parcel.md).

Content-packs wave 3 (`design/content-pack-wave-3`) ended `core`: the
state group, `ownerOf`'s state rung, `requiresCoreAccess` and the author
tier (`isAuthor`, `requiresAuthor`, the MQL permission snapshot) were
deleted; `ParcelOwner` grew the `organization` kind so the executive
(`/compact/executive`) can hold the platform's roots by manifest claim;
`heldExtents` and the widened `TreeAction` vocabulary landed; every
`can*` fails closed on an untitled path. One CI-gating lint keeps the
second fact true: `lint:untitled` (every shipped path under a claim);
`core` itself is simply gone from the source.

The core-decomposition slate (2026-08-04) retired into this doc at that
sweep; its five open questions closed as: (1) no state/commons
principal — `ownerOf` returns `null` and every caller denies, none
invents a fallback; (2) `isAuthor` is not needed once title is universal
— deleted, `heldExtents` is the within-your-extent listing; (3) the
`soul` group is PM-owned until a community holds it; (4)
`requiresCoreAccess` did not survive — deleted with both consumers
reassigned; (5) the acceptance test for "core is gone" is `lint:untitled` plus
the absence of the word from the source.
