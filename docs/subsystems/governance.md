# Governance — the Office substrate

The first substrate of the polity's structure: the **seats of
government**. An office is a *named seat with a single holder* —
apparatus-defined, authored in code, and the founder's by default until
handed off. This build models the apparatus, makes the founder the
default holder of every office (so nothing runs ownerless), lets the
founder hand a seat to another player, exposes occupancy checks + a
public roster, and wires the substrate's **first authority consumer**:
the Governor of the Central Bank controls the bank's `reserve` lever.

It does **not** build the democratic filling process (investiture,
elections, no-confidence, sortition) — that is the deferred governance
machinery. Seeded by the cooperative slate
and grounded in the [draft constitution](../governance/draft-constitution.md)
(Art. IV legislature + House names, Art. V executive / Prime Minister +
§9 administration, Art. VIII economy-is-ordinary-law, Art. XI founding
transition).

## Apparatus vs group vs chamber

Three population-shaped concepts, deliberately distinct:

- **Office** (this subsystem) — a named single-holder seat with a
  branch and an origin, **authored in code**. Not user-minted.
- **Group** ([grouping.md](./grouping.md)) — an ad-hoc, user-minted
  collection of players with roles. Any player creates one. The office
  substrate is deliberately *not* a `Group`: an office carries a branch,
  an origin, a single-holder-with-founder-default model, and
  apparatus-defined identity the group model lacks.
- **Position** ([employment.md](./employment.md)) — a slot on an
  **organization's** chart, filled by whoever holds the organization's
  appointing authority. ⭐ **The line between an Office and ordinary staff
  is whether a constitutional document points at the position.** A
  government *seat* is a `Government.seats` entry pointing at
  `(organization, positionKey)` ([civics.md](./civics.md)); an Office is
  named in `OFFICE_APPARATUS`; **personal staff is a position nobody
  points at** — a Communications Director serves at pleasure and is
  prescribed by nothing, so it is never minted as an Office. Same
  substrate throughout; the pointer is the law.
- **Chamber** ([influence.md](./influence.md)) — a population that
  *falls out of* influence (the Producer / Capital / Consumer Houses),
  never "filled". This build models only the Speaker **seats** of those
  houses, not the chamber populations.

## The Office value-object + the apparatus

`lib/governance/Office.ts` holds the value-object, its two vocabularies,
and the authored apparatus constant (the "Named value-object /
vocabulary / registry" module category — one module, one concept).

- **`Office`** — an immutable value-object: `key` (stable slug),
  `displayName`, `branch`, `origin`. Accessors only, no persistence, no
  Stuff. There is **no `cardinality` and no `mustBeFilled`** field —
  every v1 office is singular, and a vacated seat reverts to the founder
  (there is no genuine empty state, so no vacancy machinery).
- **`OfficeBranch`** = `executive | legislative` (+ `OFFICE_BRANCHES`).
  There is **no `judiciary` branch** because there is **no judiciary
  office**: a jury is a *selection from a pool*, not a seat, and that
  machinery (plus any judiciary seat) is deferred to the
  political-machinery build.
- **`OfficeOrigin`** = `constituted | founder-established` (+
  `OFFICE_ORIGINS`). *Constituted* offices are mandated by the
  constitution (abolished only by amendment); *founder-established*
  offices are stood up by the founder's pool-of-one power and are
  ordinary law (the polity may later charter, replace, or abolish them
  — Art. VIII §3).
- **`OFFICE_APPARATUS`** — the five authored seats:

  | key | displayName | branch | origin |
  |---|---|---|---|
  | `prime-minister` | Prime Minister | executive | constituted |
  | `speaker-producer-house` | Speaker of the Producer House | legislative | constituted |
  | `speaker-capital-house` | Speaker of the Capital House | legislative | constituted |
  | `speaker-consumer-house` | Speaker of the Consumer House | legislative | constituted |
  | `central-bank-governor` | Governor of the Central Bank | executive | founder-established |

  The Governor's branch is **executive** (administering the economy is
  an executive function — Art. V §9 / VIII §4) and it carries **no
  constitutional independence** (that would be a legislative choice,
  unmade); its origin is **founder-established** (ordinary law, Art. VIII
  §3) — the founder stands up a named, accountable holder over the one
  apparatus that is *live from first boot* (the central bank).

The apparatus is an authored code constant — there is no definitions
collection or YAML, and "re-seeding never clobbers occupants" is trivial
because there is nothing to seed.

## The founder default + credential resolution

The founder is the **computed default holder of every office** — the
Art. XI pool-of-one — so nothing (least of all the live-from-boot money
system) runs ownerless, with **no dependency on the founder having
logged in**. The founder is marked by **external credential**, resolved
through the `User` ↔ `GoogleProfile`/`TwitchProfile` link
(identity):

- `FOUNDER_GOOGLE_EMAIL` and/or `FOUNDER_TWITCH_HANDLE` (env, read at
  `OfficeRegistry.postRegister`). Both normalized lowercased for
  comparison; the Twitch handle matches the lowercased `TwitchProfile.login`
  case-insensitively. The original-case handle is kept for offline
  display (`founderLabel`).
- `isFounder(player)` resolves the player's Avatar → playerId → owning
  `User` (`User.find({ playerIds })`) → its `GoogleProfile.email` /
  `TwitchProfile.login`, true iff either matches. Resolved **per-check**
  (no cache) so a founder logging in *after* boot is recognized
  immediately. Returns **false until the founder has logged in** (no
  matching `User` yet) — appointment is simply inert until then, which
  is correct.

This axis is **orthogonal to** the `streamers` axis and the `isWizard`
code-trust axis ([access.md](./access.md)) — this is governance
root-power, not livestream control or TypeScript-escape capability. If
neither env var is set, `OfficeRegistry` logs one boot warning and
`isFounder` is always false (the deploy contract, mirroring absent
`WIZARD_PLAYER_IDS`).

## Sparse-handoff occupancy

`lib/governance/OfficeHolder.ts` is the persistence Document
(`office_holders` collection, `{officeKey, holderId}`, the `Group`
Document precedent with a single scalar holder). **The sparse contract:
absence of a row = the founder default holds the seat.**

- At founding the store is **empty** — every office resolves to the
  founder. A row appears only on `assign` and is deleted on `vacate`.
- `holderOf(office)` = the explicit holder if a row is stored, else the
  founder (presented by handle even while offline).
- `holdsOffice(player, office)` = explicit-holder match, OR (no explicit
  row AND `isFounder(player)`).
- `assign(player, office)` upserts the explicit-holder row (replacing any
  prior — auditable). `vacate(office)` **deletes** the row → the seat
  reverts to the founder default. `vacate` is **office-only** (no player
  arg): a singular seat reverts to the founder, there is no empty state.

## The Api / Logic surface

`OfficeRegistry` (`/obj/OfficeRegistry`, `Idea` + `PostRegistrationMixin`,
manifest-warmed) holds the durable state, mirroring `AccessRegistry` minus
all seeding. The caller surface is **the office face of `CompactApi`**
(`api/compact.ts` — the single meta-institution facade; the standalone
`OfficeApi`/`OfficeLogic` pair was folded into it in the civics build,
with office-flavored method names: `holdsOffice` / `officesOf` /
`isFounder` / `officeHolderOf` / `officeRoster` / `founderLabel` /
`assignOffice` / `vacateOffice`). The facade forwards to the
hot-reloadable `CompactLogic` singleton (`/obj/api/compact`), which owns
registry resolution + the playerId short-circuit + the fail-closed
policy. The Registry's methods carry `@CallSecurity` admitting the logic
singleton (`FromTemplate('/obj/api/compact')`) and the Api module (the
narrow-entry pattern — one state home, one calling surface, one
structurally-enforced path). (The Jul-2026 sweep briefly collapsed the
Api↔Logic tier and it was reverted — the split is the hot-reload
boundary; the fold moved the office surface BETWEEN facades, it did not
collapse the tier.)

- **Public (ungated) reads** — `holderOf` / `holdsOffice` / `officesOf` /
  `isFounder` / `roster` / `founderLabel`. The roster (offices, branch,
  origin, current holder) is **publicly readable** — governance is
  transparent by constitutional design (Art. VII).
- **Gated mutations** — `assign` / `vacate` carry the string-keyed
  `FromModule('/obj/command/governance/OfficeController')` narrow-entry
  (string-keyed to avoid a static-import cycle — the
  `AccessApi.setWizardMembership`/`WizardController` precedent). The
  **authority** is the `requiresFoundingAuthority` subcommand-level
  validator's job (see "The verb" below), not re-checked at the Api (the
  controller-gate + validator-authority split, the `WizardController`
  contract).

Gated checks derive the acting principal from execution context
(`context.commandGiver`), never a caller-supplied parameter — the
**appointer** is never a parameter; only the resolved **appointee**
playerId is.

**No-registry test path fails CLOSED.** Unlike `AccessApi.can`/`isWizard`
(which fail *open* because the dispatcher already pre-gated the
resolver), governance has no dispatcher-side pre-gate to lean on: a
missing Registry means "we cannot prove this player holds office / is the
founder", so `holdsOffice`/`isFounder` return `false`, `officesOf`/`roster`
return `[]`, `holderOf` returns `unknown`, and the mutations no-op.

## The verb

The `office` / `offices` verb (the new `governance` command category —
controller in `obj/command/governance/`, YAML in `cmd/governance/`),
afforded **universally** on `lib/character/Persona.ts` (every Avatar; the
`group`-verb precedent — NOT the operator-only `AuthorMixin`, which would
hide the public roster from non-authors):

- `offices` / `office list` — the **public** roster (any player).
- `office assign <player> <office>` / `office vacate <office>` — the
  mutating subcommands, each carrying the **`requiresFoundingAuthority`
  subcommand-level validator**. A non-founder sees the verb and the roster
  but is denied `assign`/`vacate`; the bare/`list` roster stays public
  because the gate sits on the *subcommand*, not the verb.

  `requiresFoundingAuthority` is the **single meta-level governance-root
  gate**, deliberately *not* a per-feature "is the founder" role check.
  Normal in-world authority flows through *offices* (the `reserve` →
  `requiresGovernor` / `holdsOffice` pattern below), and the founder
  exercises that authority by *holding the seats by default* — never by a
  parallel founder check sprinkled on verbs. The one power that sits
  *above* the office system is the power to constitute the government
  itself — to seat and unseat officeholders — which at founding is the
  founder's pool-of-one authority (Art. XI). That, and only that, is what
  this validator guards; its `preload` reads `CompactApi.isFounder(
  context.commandGiver)`, the actor derived from execution context (the
  `gated-api-actor-from-context` rule).

  Per-subcommand validators are a first-class command-engine feature: a
  `subcommands.<name>.validators` list fires after the verb-level
  validators and before field validators, only when that subcommand is
  invoked (see [command-routing.md](./command-routing.md) /
  [command-spec.md](./command-spec.md)). This is what lets one subcommand
  carry an authority gate the verb's public subcommands don't — added with
  this build, with `office assign`/`vacate` its first consumer.

## The one wired consumer: the Governor controls the central bank

The central bank's only control surface is the existing `reserve` verb
(`reserve mint <amount>` — the money faucet; `reserve supply` — the
audit). It was gated on `requiresWizard` (the **code-trust** axis — the
wrong axis: minting money is a monetary-authority act, not a
code-authoring one; `CentralBank.ts` / [banking.md](./banking.md) noted
its governance was left unbuilt). It is now re-gated to **holding the
`central-bank-governor` office** via the `requiresGovernor` validator
(`CompactApi.holdsOffice(giver, 'central-bank-governor')`). The Governor —
the founder by default, or a handed-off holder — controls the mint; a
wizard who is not the Governor can no longer mint. This is the office
substrate's first real consumer, the proof it works. Only `reserve`
moves; `house` (venue-owner) authority is a separate concern, untouched.

## Deferred

- The **filling workflow** (investiture-by-bill, constructive
  no-confidence, elections) — the democratic process by which the
  constitution says offices are *really* filled.
- **Juries, sortition, and any judiciary office** — a jury is a
  selection from a pool, not a seat.
- **Multi-seat bodies** (committees, boards) and any `body` cardinality.
- **Authority consumption beyond the Governor** — the PM owning
  `archwizards` (the obvious next follow-on, the seat
  [access.md](./access.md) deferred), a Speaker controlling a chamber's
  floor. v1 ships the `holdsOffice` check surface those consult but wires
  only the Governor.
- **Chambers as populations**, **terms / tenure / term-limits**, and a
  *generic* "requires office X" validator (deferred to the second
  office-gated verb — v1 uses the specific `requiresGovernor`).


## ⚠ Open: `office assign` cannot find an online player

Reported 2026-08-02, not yet diagnosed.

`office assign <player> <office>` answers **"No such player."** for a
character that is demonstrably online in a second live session.
Reproduced with a randomised handle and with a plain one (`grover`), so
it is not name tokenisation. The subcommand's `target` arg is
`type: object`, `scope: "online"`.

If it reproduces outside the e2e harness, the consequence is large: this
is governance's **only** handoff verb, so no seat can ever be filled and
every office in the game is permanently stuck on its founder default.
Everything downstream of `holdsOffice` — `requiresGovernor`, the civics
roster, the whole seats-as-positions story — would be describing a
handoff that cannot happen.

**Live consequence today.** The e2e suite needs the
`central-bank-governor` seat to exercise `reserve issue`, cannot obtain
it, and therefore runs as the **founder** instead — which holds every
founder-default seat at once. That is a blunter instrument than the
design intends and is marked INTERIM at `e2e/tests/helpers.ts`; the
intended shape (seat one ordinary character in one office) returns as
soon as this is fixed.

## History

- The build first enforced the founder gate on `assign`/`vacate`
  *inside* `OfficeController` (`OfficeApi.isFounder(commandGiver)`),
  because the command schema scoped `validators:` to the verb level —
  and a verb-level gate would have blocked the public roster. Review
  reframed this on two axes: (1) authority is an **office** concern, so
  `isFounder` should never be a per-verb gate — it is a pure internal
  predicate plus the single meta governance-root gate above the office
  system; (2) the schema limitation that forced the in-controller check
  was the signal to add **per-subcommand validators** to the command
  engine. The gate became the `requiresFoundingAuthority` subcommand
  validator, and the engine gained `subcommands.<name>.validators`
  (`feat(command): per-subcommand validators` +
  `refactor(governance): gate office assign/vacate via subcommand
  validator`).
