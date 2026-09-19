# party — the party operational core

The **party** is the operational unit that feeds combat friend-from-foe:
a named, ownable roster of combatants (players + hired mercenaries) that
fight as one side. It is the party half of the multi-party combat cycle
(the combat half is [combat.md](./combat.md)); the two are joined by one
narrow seam.

## The governing decision: a party owns its own membership

A party is **not** a managed `Group`. The grouping subsystem
([grouping.md](./grouping.md)) is a *facade* (`GroupApi`) over several
`GroupProvider` sources — most consumers (chat especially) read a
`GroupRef` audience whose membership comes from a provider, not from a
`groups` collection. The party follows that grain: it stores its **own**
roster on the `Party` Idea and registers a **fourth provider**
(`party:<path>`), so a `party:<path>` `GroupRef` resolves through `GroupApi`
exactly like `managed:<id>` — but the members come from the party, not a
minted managed group. There is exactly one membership store, and no
group↔party two-store sync.

*(This overrides the party-slate's original "back a party with a managed
Group" sketch: reaching for `GroupApi` to *store* membership is the
anti-pattern the cycle corrected — most chat membership already comes
from sources other than a managed store.)*

## The Party Idea (backed by a document)

`Party` (`lib/party/Party.ts`) is a first-class **Idea** — a live `Stuff`
in the object graph — whose state is **encapsulated on the object**:

| field | meaning |
|---|---|
| `name` | human-readable, unique-indexed |
| `founderId` / `captainId` | founder of record / current leader (succession repoints `captainId`) |
| `memberIds` | durable member refs — an Avatar `playerId`, a Mercenary `templatePath` |
| `combatSide` | the alignment key members share; `''` = the party's own `party:<path>` (the default) |
| `durable` | ad-hoc (Idea-only) vs durable (mirrored to a record) |
| `channelRef` | the party chat channel's name, or `''` |

Being an Idea (rather than a bare `Document`) buys three things: the
state lives on the object (no external store holding it); the party is
**MQL-visible** — `subscribableFields` project `name`/`memberIds`/
`captainId`/`combatSide`/`durable`, so parties are queryable by member,
side, or captain like any other `Stuff`; and it is discovered through the
**Stuff graph** (`StuffApi.findByTemplatePath`) rather than a hand-rolled
index. A runtime-minted party gets an instance `templatePath`
(`/platform/idea/party/<uuid>`), which is also its id in every `party:<path>` ref.

Its *durable* state is mirrored into a dumb **`PartyRecord`** document
(`lib/party/PartyRecord.ts`, the `parties` collection, keyed on the Idea's
`path`) — the "Idea backed by a document" shape. Membership mutations go
through `addMember`/`removeMember`; the captain is the single source of
leadership authority (`captainId`). The combat-formations build added the
**formation state**: `formationPath` (the adopted
`/platform/idea/CombatFormation/<name>` — a path string, ref-shapes Pattern
A; the party side never imports `lib/combat`) and `roleAssignments`
(member → role; **roles are sets, not seats** — many members may share
one, a departing member's role is released with them), both mirrored to
the record. The combat seam grew three pure statics beside
`sideOf`/`areAllied`: **`formationPathOf`** (the total chain — active
party's formation, else the default preset path, never null),
**`roleOf`**, and **`isCaptain`**; the captain surface grew
**`party adopt <name>`** (awaits the preset Idea resident, mints the
captain's `command` deed, and is a witnessed formation-shift beat) and
**`party assign <role> <member>`** (role names validated structurally
against the adopted formation). See
[combat-formations.md](./combat-formations.md).

## Two lifetimes over one primitive

- **Ad-hoc** (`durable=false`) — a live Idea only, never mirrored to a
  record, `StuffApi.destruct`ed when it empties, gone on restart.
- **Durable** (`durable=true`) — mirrored into a `PartyRecord` so name +
  roster + captain survive a restart, re-materialized into a live Idea by
  `PartyLogic.boot()`, and **not** destroyed on empty (it goes dormant;
  `muster` re-activates it, `standdown` sends it dormant).

A member may sit on **many** parties' rosters (`memberIds`) but has exactly
one **`activePartyPath`** at a time (the one-active-party rule, rejected at
`form`/`accept`).

## Membership on the actor: `PartyMemberMixin`

`PartyMemberMixin` (`lib/party/PartyMember.ts`, the `Employed` sparse-field
precedent) is composed on **`Avatar`** and on the hireable **`Mercenary`**
(`= PartyMemberMixin(NPC)`) — deliberately **not** the base `Character`, so
a plain townsperson or beast carries no party machinery and resolves
`solo` for free. It is a dumb store of two pointers — `activePartyPath`
(persisted) and `pendingInvitePartyPath` (transient) — written under
**participant contracts** (the first consumer of `FromClass` + `where`,
see [call-security.md § Participant contracts](./call-security.md)):
the legitimate writer is *the `Party` acting on this member* — the
active pointer may only be cleared or set to the **calling party's own
path with the member already on its roster**, the invite pointer to the
calling party's own path — plus a narrow `FromTemplate('/platform/idea/api/party')`
janitorial arm for stale-pointer cleanup when no live Party Idea exists.
The membership **transitions live on `Party` itself** (`admit` /
`extendInvite` / `release` / `recall` / `dismiss`), each owning BOTH
sides of a change (roster + member pointer) so the two stores can't
disagree; `PartyLogic` keeps only orchestration (consent checks,
persistence, channel, the empty-party terminus via
`settleAfterDeparture`, boot). `Party`'s mutation surface (transitions
+ roster/leadership/identity setters) is gated
`AnyOf(SelfOnly, FromTemplate('/platform/idea/api/party'))`; reads stay Public.
`partyMemberId()` (an Avatar's playerId, else the templatePath) is the
member's own identity answer.

## Discovery + the provider (no registry)

There is **no `PartyRegistry`** — the Stuff graph is the index. The combat
seam's synchronous read is `StuffApi.findByTemplatePath(activePartyPath)`;
the durable-party queries (`party list` / `muster`) read the `PartyRecord`
collection (indexed on `memberIds` / `name`), which is exactly the
"backed by a document gives you a queryable index" payoff. What was a
stateful singleton collapses to the **`PartyRoster` manifest singletons
`postRegister` warm** (the Api OO sweep retired `PartyApi.boot`): it registers the
`PartyGroupProvider` (`lib/party/PartyGroupProvider.ts`) with the shared
`GroupRegistry` and re-materializes durable `PartyRecord`s into live Party
Ideas. The provider is **stateless** — the id in a `party:<path>` ref is
the party Idea's `templatePath`, so it resolves straight through the graph
(`instanceof Party`), then materializes `playerId` members to online
Avatars (the `ManagedGroupProvider` shape; Mercenary templatePath members
are roster entries, not chat recipients). **Party chat** is then just a
`Channel` whose `Subject.groupRef = 'party:<path>'` (minted via
`ChatApi.createBoundChannel`, which binds the ref without minting a managed
group).

## The combat seam (the crux)

Combat asks the party subsystem exactly **two** pure, synchronous, gated
statics on `PartyApi` — and nothing else (it never touches membership, the
captain, or the roster):

```ts
PartyApi.sideOf(combatant): SideRef      // NEVER null
PartyApi.areAllied(a, b): boolean        // sideOf(a) === sideOf(b)
```

`sideOf` is a **three-rung resolution chain** (mirroring the codebase's
`ownerOf` walk):

1. **party** (de jure — Avatar / Mercenary): an active party's
   `combatSide`. Captain-settable, so "the captain sets the side" falls
   out, and "two parties ally into one side" is reachable later (point
   both `combatSide`s at one key) without reshaping the seam.
2. **owner** (de facto — pet / companion): a pet derives its side from its
   *owner's* `sideOf`. A **seam only** — pets are unbuilt this cycle;
   `sideOf` is structured to admit the rung when they land.
3. **solo**: `solo:<templatePath>` — a side of one. Two distinct solos are
   never allied, so today's 1v1 (two partyless combatants) is the
   degenerate case.

Combat reads `sideOf` **once** at session-open / join and **freezes** it on
the combatant's graph node (sides are per-session; the party persists). The
dependency is one-way (combat → party; party never imports combat), so no
cycle.

## The `party` verb + `PartyApi`

`PartyApi`/`PartyLogic` (`/platform/idea/api/party`) own the seam + the lifecycle:
`form` · `invite` · `accept` · `enlist` (the merc-hire path, no accept
handshake) · `leave` · `kick` · `disband` · `transfer` · `setSide` ·
`muster` · `standDown` · `partiesOf` · `activePartyOf`. The `party` verb
(`cmd/social/party.yaml` + `PartyController`, afforded by
`PartyMemberMixin`) is a thin translator over them.

## Cross-references

- [combat.md](./combat.md) — the consumer of the seam (multi-party fight).
- [grouping.md](./grouping.md) — the facade the `party:` provider plugs
  into.
- [chat.md](./chat.md) — party chat is a bound `Channel` (consumer, not a
  new store).
- [employment.md](./employment.md) — the `Business`/`BusinessEntity`
  mixin+entity precedent the Party/Mercenary shape mirrors.

## Deferred

Party economic/reputation facets,
party morale (Thesis 13), the client party card, coordinated party-retreat
(rout/rally) and pursuit (wayfaring), and an NPC-vs-NPC **crew** standup (a
durable-party seeder so two NPCs share a side without a live player forming
the party) — the 2v1 demonstrator (you + a recruited Mercenary vs the
duelist) exercises the full seam without it.
