# Grouping

The grouping substrate is the answer to a recurring problem in the
social cluster: there are several distinct sources of "the set of
Stuffs that share some affiliation," and every consumer that needs
to compute an audience, gate a permission, or address a cohort
should be able to ask one question — *who's in this group?* — without
caring which source is on the other end of the wire.

The three v1 sources are:

- **Managed groups** — player-created, persistent `Group` Documents
  with explicit member rosters and roles.
- **MQL queries** — read-time evaluations of an MQL query string;
  the membership IS whatever the query resolves to right now.
- **Per-Avatar contacts** — a named list stored on the owning
  Avatar via `ContactsMixin`.

These are not the same shape under the hood (one is a Document, one
is a query string, one is mixin state on Avatar). The grouping
substrate is what lets `GroupApi.membersOf` return a
`Promise<Stuff[]>` from any of them without the caller branching.

Chat is the first non-trivial **consumer** of the substrate: a
player-created `Channel` carries a `groupRef` pointing at its
membership source (a managed Group today; potentially an MQL group
or contacts list tomorrow), and the chat audience walk routes
through `GroupApi.membersOf(groupRef)`. See [chat.md](./chat.md).
The reason for the indirection — chat consumes the facade rather
than providing its own — is composition: a channel backed by "every
member of guild X" or "all my friends" works without the chat
substrate caring how membership is sourced.

Consumers in v1 are chat's audience computation and the group-DM-
to-channel promotion path. Future consumers — directed-broadcast
targeting verbs, permission gates, effect-by-cohort — plug into the
same surface without changes.

## The `GroupRef` shape

A `GroupRef` is a typed string `source:id`. The first colon separates
the provider's source key from the id; the id portion is provider-
specific and is treated as opaque by the registry.

```ts
export type GroupRef = string;

export function parseGroupRef(ref: GroupRef): { source: string; id: string };
```

The three v1 shapes:

| Source     | Ref                                       | Id portion                       |
|------------|-------------------------------------------|----------------------------------|
| `managed`  | `managed:<groupDocumentId>`               | Mongo `_id` of a `Group` doc     |
| `mql`      | `mql:<query>`                             | the MQL query string             |
| `contacts` | `contacts:<ownerPlayerId>:<label>`        | `ownerPlayerId` then label       |

The MQL case is the reason `parseGroupRef` splits on the first colon
only: a query like `mql:species:khazadicus and online` legitimately
contains further colons. The `contacts` provider does its own split
on the id portion to peel off the owner before treating the rest as
the label.

## The provider contract

Each source registers a `GroupProvider` implementation. The contract
is small enough to be worth seeing in full:

```ts
export interface GroupProvider {
  readonly source: string;
  members(id: string): Promise<Stuff[]>;
  roleOf(playerId: string, id: string): Promise<GroupRole | null>;
  isMember?(playerId: string, id: string): Promise<boolean>;
  onChange?(id: string, cb: GroupChangeListener): GroupChangeHandle;
}
```

`members` is the canonical read; `roleOf` is the canonical role
projection. Both are **mandatory** — every provider projects its
native concept into the coarse `'owner' | 'admin' | 'member'`
vocabulary, even when the underlying source has no native role
hierarchy. The contract is intentionally lossy: sources with no
role concept (`mql`, `contacts`) project every materialized entry as
`'member'`; richer hierarchies (a future guild provider with
leader / officer / member) fold their native roles into the coarse
vocabulary as best they can. `null` is reserved for "not in the
group at all." Making `roleOf` non-optional forces each provider
author to think about the projection at write time rather than
inheriting an implicit default.

`isMember` and `onChange` remain optional. When a provider doesn't
override `isMember`, the registry derives it from `roleOf` (member
iff `roleOf` returns non-`null`). `onChange` is best-effort live
notification: callers register a listener keyed by ref id, and the
provider fires it after every membership mutation. The handle's
`cancel()` removes the listener. Providers without a viable change
feed (the MQL case) return a no-op handle and document the
limitation.

Writable providers add `add` / `remove` / `setRole` / `create` /
`destroy` on top of the read contract. In v1, only the managed
provider is writable, and those mutations happen through the
provider's owning Document (`Group.addMember(...)` then
`group.save()`), not through dedicated `GroupProvider` methods —
the contract reserves the slot but v1 routes writes through the
`group` verbs and the Document directly.

## `GroupRegistry` and `GroupApi`

The registry is a singleton Idea at `/obj/GroupRegistry`, composed
with `PostRegistrationMixin`. Its `postRegister` instantiates the
three v1 providers and indexes them by source. It follows the
**catalogue / registry naming convention**: registries hold *code*
(in this case, provider implementations); catalogues hold *data*.

```ts
class GroupRegistry extends PostRegistrationMixin(Idea) {
  private providers: Map<string, GroupProvider> = new Map();

  public override async postRegister(_context?: unknown): Promise<void> {
    const managed = new ManagedGroupProvider();
    const mql = new MqlGroupProvider();
    const contacts = new ContactsGroupProvider();
    this.providers.set(managed.source, managed);
    // …
  }

  public async membersOf(ref: GroupRef): Promise<Stuff[]> {
    const { source, id } = parseGroupRef(ref);
    const provider = this.providers.get(source);
    if (!provider) throw new Error(`no provider for source '${source}'`);
    return provider.members(id);
  }
}
```

`GroupApi` is the thin caller-facing facade: `membersOf`, `roleOf`,
`isMember`, `onMembershipChange`, plus a `registry()` escape hatch
for the controllers that need to fire change notifications after
their own mutations. The Api caches a reference to the registry
Stuff on first lookup; all state lives on the registry singleton,
not on the Api. The class ends with
`SecurityApi.decorateApiClass(GroupApi)` so every external call
threads through the security gate.

`GroupRegistry.canDestruct()` vetoes destruction — it's a system
singleton; the `forceDestruct` escape hatch is the only way to
take it down.

## The three providers

### Managed (`source: 'managed'`)

Writable, persistent. Membership lives in `Group` Documents in
`Collections.Groups`. Members and roles are stored as two
index-aligned parallel arrays — `memberIds: string[]` and
`memberRoles: ('owner'|'admin'|'member')[]` — over the marshalled-Map
alternative. The shape honors the persistence rule's preference for
scalars and arrays of scalars, and makes Mongo queries like
`{ memberIds: someId }` trivial.

`members(id)` loads the Group by `_id`, then materializes each
playerId through `PlayerApi.findAvatarByPlayerId`. The materialization
is an **online filter** — offline members materialize to nothing.

The provider also exposes a by-name lookup:

```ts
async findByName(name: string): Promise<Group | null>;
```

`name` is unique-indexed at the collection level, so the result is a
single Group or `null`. Used by `AccessRegistry`'s bootstrap seeding
and developer-cache warm path to find well-known groups (`'core'`,
`'lounge'`, `'wizards'`) without threading their `_id`s — see
[access.md](./access.md).

The provider holds a `Map<id, Set<listener>>` for change notifications.
The `fireChange(id)` method is called by `GroupController` after every
CRUD mutation (`make` / `delete` / `rename` / `add` / `remove` /
`role`).

### MQL (`source: 'mql'`)

Read-only. The ref's id portion IS the query string. `members(id)`
delegates to `MqlApi.resolveMany(query, { commandGiver, scope: 'online' })`
and returns the result set.

The provider takes the first registered Avatar as the commandGiver —
the resolver requires one and the `online` scope semantics don't
depend on which viewer drives the read. If no Avatars are online,
the result is empty (the resolver can't run without a viewer).

`onChange` is a documented v1 limitation. The `mql-subscription`
substrate keys live subscriptions to an `Interactive`, which a
server-side group consumer doesn't have. Three resolutions exist
in the risk doc (synthetic Interactive / lower-level resolver seam /
polling); the v1 choice is **no live updates**. Listeners install
cleanly and silently never fire. Callers wanting reactive MQL-backed
membership poll explicitly.

### Contacts (`source: 'contacts'`)

Read-only, model-B per-Avatar. Storage lives on `ContactsMixin` as
a `Set<ContactEntry>`; the provider is a viewer over that state.

The ref shape is `contacts:<ownerPlayerId>:<label>`. `members(id)`
splits on the first colon of the id portion to peel off the owner,
finds the owner's Avatar by playerId, and resolves each
`ContactEntry` according to its kind:

- `avatar` entries → `PlayerApi.findAvatarByPlayerId(playerId)` (online filter).
- `npc` entries → `StuffApi.findByTemplatePath(templatePath)`
  (only currently-cloned NPCs surface).

**Owner-only privacy** is enforced at the provider boundary, not at
the verb side. The provider reads the current command's giver from
`ExecutionContextApi.getCurrentCommandContext()` and returns an
empty set if the asker isn't the owner. Programmatic calls with no
command context bypass the gate — only the verb path is privacy-gated.
Placing the check on the provider means any cross-cutting consumer
(audience computation in chat, future targeting verbs) gets the
same privacy guarantee for free.

## The `Group` Document

The managed provider's persistent shape:

```ts
export class Group extends Document {
  static collectionName = 'groups';
  static fieldMeta: FieldMeta = {
    name: { persistent: true },
    owner: { persistent: true },
    memberIds: { persistent: true },
    memberRoles: { persistent: true },
  };

  name: string = '';
  owner: string = '';                  // owner's playerId
  memberIds: string[] = [];
  memberRoles: GroupRole[] = [];

  addMember(id: string, role: GroupRole = 'member'): boolean;
  removeMember(id: string): boolean;
  setMemberRole(id: string, role: GroupRole): boolean;
  roleOf(id: string): GroupRole | null;
  isMember(id: string): boolean;
}
```

The methods are the only external mutation surface — direct writes
to `memberIds` or `memberRoles` bypass the invariant that keeps the
two arrays index-aligned. `addMember` rejects duplicate ids;
`setMemberRole` rejects unknown role tags; both throw on invalid
role vocabulary rather than silently coercing.

The role vocabulary is the coarse common set: `'owner' | 'admin' |
'member'`. Source systems wanting richer hierarchies model them
internally — the facade returns the coarse role.

## The `group` verb suite

`GroupController` is the player-facing surface for the managed
provider. The YAML lives at `mud/cmd/social/group.yaml`; the controller
dispatches on the subcommand:

```
group make <name>
group delete <name>
group rename <old> <new>
group add <name> <target>
group remove <name> <playerId>
group role <name> <playerId> <owner|admin|member>
group show <name>
group list
```

Every mutation that touches a Group fires
`GroupRegistry.managed().fireChange(g._id)` so subscribed consumers
update. The verbs gate on ownership (`delete`, `rename`, `role`) or
owner-or-admin (`add`, `remove`) by reading the Group's role of the
acting Avatar's playerId.

The suite is **exposed via composition on `ContactsMixin`**, which
declares `commandContributions.self = ['social/contacts.yaml', 'social/group.yaml']`.
ContactsMixin composes on every Avatar; the `group` verbs ride along
because both are personal-list management verbs an animate
being uses to organize who-they-know. NPCs that compose
`ContactsMixin` in their own content gain both verbs uniformly.

The verb surface is **for managed groups only**. There is no
`group make mql:...`, no contacts-creating subcommand, no
channel-creating subcommand. The other sources are consumed
through their own paths: MQL queries are typed at call sites,
contacts are managed through `contacts add` / `contacts remove`,
channels through `chat make` / `chat promote`. `group list` lists
the caller's managed Groups; the other sources are introspected
through their owning subsystem's verbs.

`chat make` mints a backing managed Group for each player-created
channel — the chat substrate is the **owner** of those Groups,
and the user-facing `group list` view filters them out by
consulting `ChatApi.getBackingGroupIds()`. The Group model itself
knows nothing about chat; ownership of "which groups did chat
spawn?" lives on the consumer side, not on the substrate.

## Future directions

Several layered features were scoped out of v1, named here so the
grouping substrate's shape stays legible to future builds:

- **Predicate / dynamic groups beyond the MQL provider.** The
  MQL provider already covers the "membership is a live query"
  case. A richer predicate language with cached recompute, defined-
  by-author segments, and a non-`Interactive` resolver seam is the
  natural Layer 2 evolution.

- **Nested / composite groups.** A "group of groups" — a faction of
  guilds, all my sections, the union of two managed teams. Defers
  until a consumer actually needs it. The facade is the right place
  to compose; concrete providers stay flat.

- **Shared grant primitive.** A `(GroupRef, role) → capability`
  pattern that would let multiple consumers share their role →
  permission mapping rather than each writing its own. v1's chat
  audience computation and group-DM promotion are not enough
  consumers to justify the extraction. The right time is when a
  third consumer (a permission gate, a targeting verb) repeats the
  shape.

The recognition family — per-bucket notification policy, display
lensing per viewer, recognition state — stays in
`social-graph-slate.md` for the recognition build. Grouping
supplies membership and the coarse role; recognition layers
viewer-dependent semantics on top.

A group can additionally BE a **committee** — the meta-layer role a
managed group plays when it holds parcel title over a subdivision
(derived, never flagged: [access.md](./access.md) § The committee;
reads on `CompactApi`). Committee chat channels bind the committee
group's ref like any bound channel — the substrate is unchanged.

## Related

- [chat.md](./chat.md) — grouping's first multi-shape consumer;
  channel membership routes through `GroupApi.membersOf(groupRef)`.
- [contacts.md](./contacts.md) — `ContactsMixin`'s storage shape and
  CRUD verbs; the substrate the contacts provider views over.
- [messaging.md](./messaging.md) — `MessageApi.scene(...)` and the
  Sensor delivery chokepoint; audience computation feeds into Scene
  composition.
