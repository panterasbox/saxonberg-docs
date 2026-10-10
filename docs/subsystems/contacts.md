# Contacts subsystem

Per-Avatar named lists of other characters — the prototypical
**model-B** `GroupProvider` source. A contact list is a *facet of the
owner Avatar*, not a primary entity with members of its own. Labels
(`"friends"`, `"study-group"`, `"ignore"`, anything else the owner
chooses) are derived views over a flat entry set; there is no
reserved label vocabulary, and `friend` carries no special status.

Composed onto `Character` (and so onto every `Avatar`), `ContactsMixin`
ships the storage + method surface; `ContactsController` + `contacts.yaml`
ship the player-facing verb suite; `ContactsGroupProvider` plugs the
storage into the broader `GroupApi` so future audience-computation
consumers can reach a contacts list with the same uniform shape they
reach a managed group with.

## What contacts is — and isn't

| Use this | When membership is | Example |
|---|---|---|
| `ContactsMixin` (this subsystem) | a facet of one Avatar — *who do I know* | a player's `friends`, `study-group`, `ignore` lists |
| Managed `Group` Document | a first-class entity with its own identity + roster | a guild's roster; a course cohort |
| MQL provider | a query result — *whoever currently matches* | "everyone in the cafeteria with `species:khazadicus`" |

The model-B framing is load-bearing. A managed `Group` lives in its
own Mongo document and has members like a guild has members — the
group is the thing. A contact list is the owner's *view* of other
characters; outside the owner Avatar it doesn't really exist as a
discrete object, even though `GroupApi` exposes a uniform handle for
it.

This is also why contacts ships with no `make` / `delete` verbs for
labels: labels are derived (the set of distinct `label` values across
the entries). Creating an empty `study-group` label without any
entries would be naming something that doesn't yet exist.

## Entry shape — durable identifiers only

```ts
export type ContactEntry =
  | { kind: 'avatar'; playerId:     string; label; source; addedAt }
  | { kind: 'npc';    templatePath: string; label; source; addedAt };
```

Two kinds, distinguished by `kind`. The discriminator field across
both kinds is a **durable identifier**: `playerId` for Avatars (stable
across name changes, character edits), `templatePath` for NPCs (since
NPCs are runtime clones whose instance ids are ephemeral).

There is **no `user` kind in storage**. The multi-character expansion
implied by `contacts add iffy friend` happens at the controller layer
as add-time sugar (one `avatar` entry per playerId of the resolved
User); the persisted shape never mentions the User concept. Removing
one of those characters is just removing one entry.

The `source` field is reserved for the future
**faction-pushed-roster mechanism** — e.g., `source: 'guild:gamma'`
on entries the guild's roster pushes down, with semantics like "owner
can hide locally but can't remove individually." v1 always writes
`'self'`. `addedAt` is a ms-since-epoch timestamp captured at add
time, useful for stable display ordering.

## `ContactsMixin` shape

`ContactsMixin` is composed onto `Character` so every Avatar carries
it natively. NPCs that want their own contact list compose it
explicitly in their content class.

```ts
export interface Contacts {
  addContact(entry: ContactEntry): boolean;
  removeContact(kind: ContactEntry['kind'], ref: string, label: string): boolean;
  contactsByLabel(label: string): readonly ContactEntry[];
  contactLabels(): string[];
  clearContactLabel(label: string): number;
  renameContactLabel(oldLabel: string, newLabel: string): number;
  allContacts(): readonly ContactEntry[];
}
```

`addContact` returns `false` on a duplicate (same `kind` + ref +
`label`) so the controller can report "already in list" without a
second probe; the underlying storage stays append-only on the success
path. `contactLabels()` returns the distinct label set, sorted.
`renameContactLabel` mutates every entry in-place — a non-destructive
relabel.

Storage is a single persistent field:

```ts
static fieldMeta: FieldMeta = {
  _contacts: { persistent: true },
};
_contacts: ContactEntry[] = [];
```

Persisted as plain JSON via reflection through the Hydrator. This
departs from the "scalars only" default for persistent fields and
follows the precedent set by `AliasMixin.aliases` (a
`Record<string, string|null>` stored directly without a marshaller).
The shape is regular enough — discriminated union of two flat records
of scalars — that the marshaller escape hatch isn't earned. The
default `[]` mirrors AliasMixin's legacy-tolerant pattern: existing
Avatar docs without the field hydrate cleanly to an empty list.

`ContactsMixin` contributes the `contacts` and `group` verbs through
its `commandContributions.self` array (see
[command-routing.md](./command-routing.md)) — they ride on every
Contacts-bearing being.

## The `contacts` verb suite

```yaml
verbs: [contacts]
controller: ContactsController
validators:
  - /lib/command/validators/requiresAnimate
subcommands:
  add:
    options:
      char: { short: c, type: boolean }
    args:
      - { name: target, type: object, required: true, scope: "online" }
      - { name: label,  type: string, required: true }
  remove: { args: [ { target: string }, { label: string } ] }
  show:   { args: [ { label:  string } ] }
  list:   {}
  clear:  { args: [ { label:  string } ] }
  rename: { args: [ { old_label: string }, { new_label: string } ] }
```

The `add` subcommand resolves its `target` via MQL at `scope: online`
(the v1 limitation). `remove` and friends take raw strings — `remove`
matches against entries already in the caller's list, so it doesn't
need to re-resolve through MQL.

The `--char` flag on `add` opts out of the default multi-character
expansion. Without the flag, the controller expands; with it, only
the single resolved Avatar's playerId is stored.

## Multi-character expansion at add-time

The default behavior of `contacts add iffy friend`:

1. Resolve `iffy` online via MQL → a single Avatar Stuff.
2. Walk `iffy.getUser()` → the User document → `user.playerIds`.
3. For each playerId, append one `kind: 'avatar'` entry with that
   playerId and the specified label.
4. Feedback prose joins the *character names* of every added Avatar,
   bulk-fetched from their templates.

With `--char`, step 2-3 collapses to a single `addContact` for the
matched Avatar's own playerId.

The literal word **"user"** never surfaces in player-facing prose —
feedback is framed entirely as character names (`"Added Iffy, Spark,
and Bramble to your friend list."`). The User concept is an internal
aggregation, not a thing the player addresses; sibling-character
expansion is presented as adding several characters at once.

## Show / list / remove semantics

`contacts show <label>` materializes the stored list by **bulk-fetching
Avatar Templates** for the avatar entries and NPC Templates for the
NPC entries:

```ts
const paths = playerIds.map((id) => Avatar.TEMPLATE_PATH_PREFIX + id);
const templates = await Template.findByPaths(paths);
```

Each Avatar shows as its own identity — no User-level grouping, no
"canonical name" selection. If a player added all three siblings of
a friend's account, all three show, each on its own line. NPCs render
their template name with an `(NPC)` suffix.

`contacts list` returns the distinct labels currently in use, with
entry counts. No `make` / `delete` for labels — once the last entry
of a label is removed, the label drops out of `list` automatically.

`contacts remove <target> <label>` matches against entries **already
in the caller's list**. It does not do a global search. The match is
case-insensitive against the stored `playerId` (for avatar entries)
or `templatePath` (for npc entries). This makes `remove` cheap and
keeps it from accidentally targeting an unrelated Avatar.

`contacts clear <label>` bulk-removes every entry of a label;
`contacts rename <old> <new>` is a non-destructive relabel.

## Owner-only privacy

Privacy is enforced at the **provider boundary**, not the verb. The
`ContactsGroupProvider.members(id)` lookup walks
`ExecutionContextApi.getCurrentCommandContext()` and refuses the read
if the asker isn't the owner:

```ts
const ctx = ExecutionContextApi.getCurrentCommandContext();
if (ctx) {
  const asker = ctx.commandGiver as Stuff;
  if (asker !== owner) return [];
}
```

When there's no command context (a programmatic call from substrate
code), the read is permitted — the gate exists to keep one player's
verbs from peeking at another's lists, not to lock the subsystem
against its own engine. Per-list visibility tuning is deferred to the
recognition-family build.

## GroupApi integration

`ContactsGroupProvider` is registered against `GroupApi` with
`source = 'contacts'`. The ref shape is:

```
contacts:<ownerPlayerId>:<label>
```

So `contacts:abc123:friend` resolves to "the `friend` list belonging
to player `abc123`." `GroupApi.membersOf` dispatches to the provider,
which materializes the entries:

- `avatar` entries → live Avatar via
  `PlayerApi.findAvatarByPlayerId` (online filter — offline siblings
  silently surface as nothing in the materialized view, though they
  remain stored).
- `npc` entries → live NPC via `StuffApi.findByTemplatePath` if the
  NPC is currently cloned; otherwise nothing.

This means any future audience-computation consumer that knows how
to talk to `GroupApi` — broadcast with `--to` against a contacts
label, a future `dm @friends` shape, an effect targeting a player's
study-group — reaches the contacts list uniformly with managed groups
and MQL queries. No per-consumer special case.

`onChange` registration is also routed through the provider:
`ContactsController` calls `fireChange(ownerPlayerId, label)` after
every CRUD mutation, and subscribed listeners (chat audience cache,
future targeting subscriptions) get a tick.

## Online-only `contacts add` — v1 limitation

The `add` subcommand resolves `target` at `scope: online`. Targeting
a *currently logged-out* character by typed name alone isn't
supported in v1; the workaround is timing — add the friend while
they're online once and the playerId is durable past that point.

Offline targeting is deferred to a future **encounter-memory**
mechanism (sketched as `EncounteredMixin` in the build plan's
deferred-design section): every Avatar keeps a per-Avatar record of
recently-encountered other characters (name + playerId, multi-day
TTL) so `contacts add` on an offline target can fall back to that
local memory instead of needing a global online resolution. The
persisted shape doesn't change when that lands — storage is already
keyed on durable identifiers; only the add-path's resolver gains a
fallback lookup.

## The attention layer above contacts — see social-graph

`ContactsMixin` ships the **bucket primitive** — named lists of
durable identifiers, plus the verbs to manage them, plus a uniform
read surface via `GroupApi`. The **attention-management layer** that
consumes those buckets — per-rule **notification policy** and
**display lensing** — shipped as social-graph Wave 3; see
[social-graph.md](./social-graph.md). Notably it does **not** put the
policy on contacts: a rule is keyed on **any `GroupRef`** (a contacts
label is one kind of subject, alongside managed groups and MQL
queries), stored as a separate ordered `NotifyPolicyMixin` list and
driven by the new `notify` verb — so contacts stays the dumb storage
primitive it was designed to be. Resolved there:

- **Per-rule notification policy** (`onConnect` / `onDisconnect` /
  `onMessage`; movement deliberately dropped) — the `notify` verb +
  the login presence relay.
- **Display lensing** — bucket-keyed room-occupant verbosity (the
  formatter is a sibling of `describeFor`, *not* a
  "DescribeApi v2").
- **Reserved labels** — `friends` / `foes` / `everyone-else` /
  `strangers` are now first-class virtual baseline rules with shipped
  defaults (contacts labels themselves stay generic strings).

Still deferred (now tracked in social-graph.md + its slate's Wave 4):

- **Recognition state** as a separate "do I know this person yet"
  axis distinct from bucketing, and **mutual / consent friending**
  (`friend` adds stay one-sided).

The contracts hold across the upgrade: the persisted entry shape
gains optional fields; the existing verbs keep working; the
`GroupApi` ref shape stays the same.

## Related

- [grouping.md](./grouping.md) — the `GroupApi` facade,
  `GroupProvider` interface, managed/MQL/contacts provider trio;
  this is where the model-A/model-B split lives.
- [messaging.md](./messaging.md) — audience computation in
  chat / DM; the consumer side of `GroupApi.membersOf` against a
  `contacts:` ref.
- social-graph-slate.md — the
  full recognition-family design (notification policy, display
  lensing, recognition state) that contacts is the first slice of.
