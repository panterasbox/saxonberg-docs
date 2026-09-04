# State Model

This doc describes the data model for game-world state: how
hydration, persistence, and identity fit together for `Stuff`. The
companion docs cover the mechanism:
[templates.md](./templates.md) for the clone pipeline,
[lifecycle.md](./lifecycle.md) for the create/destroy choreography,
[persistence.md](./persistence.md) for the auth-record track that
deliberately doesn't go through any of this.

## The Unified Model

**Hydration** = CMS template `data` → backing, at clone time.
**Persistence** = backing → CMS template `data`, at destroy/shutdown.
**Reset** = re-hydrate an existing backing back to template defaults.

(*Reset* is not implemented yet; *persist* ships for Avatar — see
§ Persist Direction — v1.)

These are one mechanism with two directions. Both use
`MixinApi.getAllPersistentFields(backing.constructor)` as the single
source of truth for which fields round-trip. The standard
`PersistentHydrator` walks that list and copies between `data` and the
backing. Any future "persist" implementation will walk the same list in
the opposite direction.

### Shutdown-save IS inverse hydration

Every Stuff at a path serializes back to its own template doc. On
restart, the clone pipeline rehydrates. The template IS the save
state. There is no separate "save state" subsystem, and there will
never be one — the symmetry is the point.

## Two Categories of Stuff

- **Self-contained**: state lives at one path, in one doc. Hydrate ↔
  persist are perfectly symmetric over the same field set. Rooms,
  doors, props, most NPCs, **and avatars**. The standard
  `PersistentHydrator` handles them.

- **Reference-following** (rare): hydration traverses pointers to other
  Stuff first, then assembles state. The avatar was originally assumed
  to be one of these (with separate `Player` and `CharacterSheet`
  records); the unified model collapsed that complexity. Avatar is now
  self-contained.

Reference-following Stuff — if any survive — gets a custom `Hydrator`
subclass. See [templates.md § The Hydrator Contract](./templates.md#the-hydrator-contract)
for that escape hatch.

## The Data Model

### User is a Document (a plain record, not a Stuff)

User is **meta-game**, not game-world: it's auth-layer (identity,
authentication, session context). It doesn't live in a zone, doesn't
have a `templatePath`, doesn't get cloned, doesn't hydrate, doesn't
participate in the Stuff filesystem.

It is **not** in the Stuff hierarchy at all. `User extends Document` —
a plain MongoDB-backed record with no `stuffId`, no registry
membership, no proxy, and no lifecycle. Construction is a plain
`new User()`; loading via `User.findById(...)`. Lookup by Mongo `_id`
or `googleProfileId` via the inherited `find`/`findById` statics. It's
value-like: two `findById` calls for the same id return distinct
instances.

See [persistence.md](./persistence.md) for the `Document` contract
in detail.

### User owns its list of characters

```typescript
class User extends Document {
  static collectionName = 'users';
  static fieldMeta: FieldMeta = {
    googleProfileId: { persistent: true },
    playerIds: { persistent: true },
  };
  googleProfileId: string = '';
  playerIds: string[] = [];
}
```

This intentionally violates the general "avoid bidirectional arrays"
guideline, because the cost factors that guideline targets don't apply:

- The list is **bounded** (a user owns ~1–10 characters).
- It's **low-churn** (mutates only on character create/delete).
- The authoritative answer to "which characters does this user own?"
  belongs on User. Scanning the avatar collection for ownership would
  be semantically backwards.

Character creation appends to `user.playerIds` and captures the first
persistence-spine snapshot (**no per-player template row** — the
identity doctrine, ref-shapes.md; a legacy pre-retirement character's
row is read once as a fallback, then superseded by its snapshot).
Character deletion removes the id and the snapshot. Two writes on rare
events; not atomic, acceptable.

### No Player class

`Player` was a join row with two fields (`userId`, `characterSheetId`).
Under the unified model it earned no keep — its state was entirely
pointers to other records. Its responsibilities dissolve into:

- "Which characters does this user own?" → `user.playerIds`
- Character state → the Avatar's persistence-spine snapshot

The id is **still called `playerId`**. It represents "one of a user's
owned character slots" — that meaning survives the class's death.
Paths use `/platform/agent/Avatar/<playerId>`.

### No CharacterSheet class

`CharacterSheet` existed to hold persistent character state across
clones. Under the unified model, the Avatar's persistence-spine
snapshot (keyed on the minted identity path `/platform/agent/Avatar/<playerId>`)
holds that state directly. The sheet would be indirection for a problem
the unified model already solves.

### Avatar is self-contained

Avatar's template doc carries every mixin-declared persistent field as
`data`. No pointers to Player, no pointers to Sheet. The standard
`PersistentHydrator` fully populates the avatar at clone time:

```js
{
  path: "/platform/agent/Avatar/<playerId>",
  class: "/platform/agent/Avatar",
  hydratorClass: "/platform/idea/persistence/PersistentHydrator",
  data: {
    name, surname, honorific, nameSuffix, alternateNames, fullName,
    pronouns,
    shortDescription, longDescription,
    keywords,
    // …every mixin-declared persistent field
  }
}
```

Runtime-only pointers (`user`, `interactives`) are NOT in
`fieldMeta`'s persistent entries. They're stamped from the clone context (`user`) or
established as connections come and go (`interactives`).

## The Clone Pipeline (Brief)

Full pipeline is in [templates.md § The Clone Pipeline](./templates.md#the-clone-pipeline).
The state-relevant ordering:

1. Construct (sentinel-gated).
2. Stamp `zone` (from `ZoneApi.resolveZoneForPath`).
3. Stamp `templatePath`.
4. Wrap in Proxy.
5. **Register** in `StuffApi.objectsById`.
6. **Hydrate** if `hydratorClass` is named — `PersistentHydrator`
   runs its two-phase dispatch: Phase 1 walks
   `MixinApi.getAllPersistentFields(constructor)` and prefers
   `await target.set<Field>(value)` (falls back to bracket-assign);
   Phase 2 walks `MixinApi.getAllInstructionFields(constructor)` and
   calls the required `await target.apply<Field>(value)`. See
   [templates.md § The Hydrator Contract](./templates.md#the-hydrator-contract).
7. **`postRegister(context)`** if the backing composes
   `PostRegistrationMixin`.

Register-before-hydrate is load-bearing: the in-flight object must be
resolvable by `stuffId` so that hydrators with self-references find it.

If hydrate or `postRegister` throws, the object is unregistered before
the error propagates.

## The Context Bag

`StuffApi.clone(path, context?)` accepts an opaque `context` bag for
runtime-only state that can't come from the template — typically
references to other runtime objects.

Avatar is the canonical example:

```typescript
// Avatar.ts
export interface AvatarInitContext {
  user?: User;
  playerId?: string;
}

class Avatar extends AvatarBase {
  user?: User;        // runtime-only, set in postRegister
  playerId: string = '';

  override async postRegister(context?: AvatarInitContext): Promise<void> {
    if (context?.user) this.user = context.user;
    if (context?.playerId) this.playerId = context.playerId;
    if (this.playerId) PlayerApi.registerAvatar(this);
  }
}

// Login flow:
const avatar = await StuffApi.clone<Avatar>(
  Avatar.getTemplatePath(playerId),
  { user, playerId }
);
```

`avatar.user` is re-established every login from the context. It is
NOT in `fieldMeta`'s persistent entries — User isn't part of Avatar's persistent
state (ownership lives on `User.playerIds`).

The context is `unknown` at the API surface. Subclasses narrow it to
their concrete type locally rather than threading a generic through
`StuffApi.clone`.

## Paths and Collections

Single `domain` collection, path-namespaced:

- `/platform/… + /stuff/[<sub>/]<ClassName>` — singleton class templates
  (`/platform/idea/EventRegistry`, `/platform/idea/hooks/DomainHook`).
- `/platform/… + /stuff/[<sub>/]<ClassName>/<id>` — multi-instance class templates
  (e.g. `/platform/agent/Avatar/<playerId>`).
- `/world/...` — content (rooms, doors, props, NPCs).

Folder/leaf invariant on `domain`: Zone templates may have descendants;
non-Zone templates may not. Enforced by `DomainHook` against the PM
chokepoint. See [templates.md § TemplateApi & the Folder/Leaf Invariant](./templates.md#templateapi--the-folderleaf-invariant).

`users` and `google_profiles` stay as their own collections — they're
`Document` records, not templates. Their lifecycle (plain `new` to
construct, `findById`/`find` to load, `save`/`delete` for the DB
round-trip) is independent of the template/clone pipeline.

## Stamped-on-Stuff Fields

Every Stuff carries:

- **`stuffId: string`** — runtime ID, generated in the base
  constructor.
- **`templatePath: string | null`** — the clone path, or `null` for
  `create`/`createSync`. Used by identity-keyed security policies
  (`FromTemplate`).
- **`zone: Zone | null`** — universal subdivision. Stamped at clone
  time from the nearest-ancestor Zone template; NOT back-filled by
  `ContainmentApi.move` — zone identity follows whichever template
  spawned the item, not whichever container it currently sits in.
  Runtime-only for now: Zone references are not auto-persisted
  (mirrors how `inventory`/`environment` work — the authoritative
  source for zone membership is the `domain` template path at clone
  time).

These are stamped by `StuffApi.clone`, not declared on subclasses. They
exist on every Stuff regardless of mixins.

## Things the Model MUST NOT Regress

- **Multiplexing**: multiple Interactives can connect to the same
  Avatar. Routing is via `ConnectionApi.transfer(interactive, target)`
  / `detach(interactive)`; the connection storage lives on
  `HasInteractiveMixin` (composed by both `Avatar` and `Login`).
  Interactive itself knows nothing about Avatars — see
  [architecture.md](../architecture.md) for the HasInteractive vs Avatar
  distinction.
- **Character selection**: multi-character flow is structurally
  supported via `user.playerIds`.
- **PlayerApi registration**: still happens, just from `postRegister`
  now (`PlayerApi.registerAvatar(this)`).
- **Avatar destroy/cleanup**: still unregisters with `PlayerApi`, still
  drops all `interactives`.
- **Auth flow**: Google OAuth → User/GoogleProfile find-or-create
  unchanged. The only diff is "Player creation" → "Avatar template
  creation".
- **Hydrator opt-in**: `hydratorClass` is opt-in. Templates that omit
  it skip hydration entirely.

## Persist Direction — the self-persistence spine

Avatar persists through the **universal self-persistence spine** (a general
substrate shared by avatars, rooms, and vessels) into its own
`holder_snapshots` record — NOT its template. See
[persistence.md § The self-persistence spine](./persistence.md#the-self-persistence-spine-persistable).

- **`Avatar.save()`** → `PersistableApi.capture`: captures the avatar's
  declared fields, **carried inventory** (Container slice), **worn gear**
  (Slotted slice), and **spawn/recall location** (`place`, WarrenMember-
  reconciled). The synchronous snapshot prefix reads state BEFORE the first
  await; the MongoDB write is async.
- **`Avatar.restore()`** → `materialize`: re-hydrates a live instance from
  its record. v1 is developer/admin only (and intended for a fresh instance).
  The normal login path materializes automatically via `postRegister`.
- **Auto-save fires** on logout/linkdead (via `Avatar.onDestruct`'s
  fire-and-forget save) and on a periodic backstop installed by
  `Avatar.enter()` (called from `Login.enter` after the connection
  transfer). Cadence comes from the `world.autosave.interval`
  setting (default 5 minutes); per-Avatar overrides fall out of the
  standard `resolveSetting` chain.

Now in scope (the migration onto the spine): **inventory + worn gear
persist** (the "no naked login" fix — items were lost on restart before) and
the substrate is **general** — an authored persistable room persists its
contents the same way. Still out of scope for v1:
- **Multi-process coordination** — concurrent saves resolve as
  last-write-wins via MongoDB's `replaceOne`. Cross-process locking
  would belong at the Mongo client or lifecycle layer if/when a
  multi-shard topology materializes.
- **Reset** — re-hydrate an existing backing back to template
  defaults. Not built; the unified model leaves room.
- **`/state/`** as a separate path namespace from `/world/` —
  implement if/when the distinction becomes load-bearing.

## Cross-References

- [templates.md](./templates.md) — clone pipeline, `Hydrator` contract,
  `PostRegistrationMixin`, `TemplateApi`, folder/leaf invariant
- [lifecycle.md](./lifecycle.md) — construction sentinel, ProxyApi
  wrap, synthetic constructor frame, destruct → canDestruct veto
  → onDestruct witness → cleanupOnDestruct walk → shadow detach
  → destroy → unregister
- [persistence.md](./persistence.md) — the `Document` track for
  auth/meta records (User, GoogleProfile); around-save/delete hooks
- [call-security.md](./call-security.md) — `templatePath` stamping
  feeds `FromTemplate` policies; `Stuff.destroy` decorator stack
- [antipatterns.md § Per-Field Invariants](../antipatterns.md#per-field-invariants-belong-on-setters-not-in-normalize-hooks)
  — setter-based invariants that hydration rides on
