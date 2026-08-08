# Combat formations

The **party-strategy layer over the combat engine**: a *formation* is a
named, standing, party-level **policy over the threat graph** — who holds
which engagement, who intercepts what, who finishes the fallen — chosen
once by the party captain (`party adopt <name>`) and then *watched
unfolding*. Set-policy-then-watch is the text-native answer to the
per-character gambit wall (DA:O's "Tactics" menu — the model this build
deliberately rejects; hence the name *formation*, the word that describes
what this actually is). The formation never scripts anyone's gambits:
moment-to-moment play (strike/feint/defend/flee) stays personal agency;
the formation is the standing policy around it.

Built by the combat-formations build; seeded by
combat-tactics-slate.md
as refined by combat-slate.md
(the emergent Master-Apprentice validation + coup attribution). Rides
[combat](./combat.md) (the `CombatGraph` melee), [party](./party.md) (the
captain + roster), [advancement](./advancement.md) (`ActSignature`
credit), and [accountability](./accountability.md) (the death-row facts).

## The `CombatFormation` Idea

`lib/combat/CombatFormation.ts` — a singleton Idea per preset at
`/obj/CombatFormation/<name>` (the `LocomotionMode` shape: authored
data, no registry), seeded from `seeds/lib/combat/CombatFormation/`. The
Idea carries only **declared shape** — role vocabulary, allocation kind,
interception rule, coup governance; what a token *means* is combat's
module-private interpreter in `CombatLogic`, and every magnitude is a
`combat.formation.*` AppSettings dial (shape in code, magnitudes in
dials). `policy()` returns a frozen, memoized value (invalidated by every
setter, so a CMS re-hydrate lands on the next consult).

The four presets:

| Preset | Policy | Coup |
|---|---|---|
| `default` | sustained targeting, no interception | the engaged attacker's |
| `focus-fire` | `called` — converge on the captain's target | captain's call |
| `vanguard` | `front`/`back` roles; any edge onto a `back` intercepts to a `front` | the engaged attacker's |
| `master-apprentice` | `primary` — the side presses the apprentice's target; the master intercepts at high-threat | the **apprentice's** right, the captain's call |

## The total resolution chain — "solo" is not a concept

Formation resolution is **total**: `PartyApi.formationPathOf(combatant)`
returns the active party's chosen formation path, else
`DEFAULT_FORMATION_PATH` (`/obj/CombatFormation/default`) — never
null, never `''` (the `sideOf` solo-rung / locomotion universe-`walk`
mirror). A partyless wanderer, a party that never chose, and a party of
one all resolve through the same read; there is **no `if (formation)`
branch anywhere in the exchange loop**. Combat resolves the path to its
Idea per beat (`resolveFormationFor` — deliberately NOT frozen at open,
unlike `side`, which is what makes a mid-fight `party adopt` land on the
very next beat with no pending-state machinery); when the Idea is not
resident (bare tests, cold boot) the built-in
`CombatFormation.DEFAULT_POLICY` fallback applies, pinned value-equal to
the seeded default. **The default preset byte-preserves pre-formation
combat** — a gym regression pins four canonical matchups' exact winners
and beat counts.

The seam stays narrow and one-way: combat imports only `PartyApi`
(`sideOf`/`areAllied` + the new `formationPathOf`/`roleOf`/`isCaptain`
statics); the party side stores and returns **path strings only**
(a ref-shapes identity ref) and never imports `lib/combat` — role names are
validated structurally (a duck-read `getRoles()`).

## Roles are sets, not seats

A role is an open badge the captain assigns (`party assign <role>
<member>`): any number of members may hold the same role, each member
holds at most one, and there is no cardinality schema or composition
table. Two rules interpret any fill deterministically: a **vacant**
role's clauses are inert (the policy degrades to its fallback rung — a
Vanguard with no front simply stops intercepting), and a **plural** role
resolves to its holders in **party-roster order** wherever a clause needs
exactly one referent (the interception walk; MA's primary apprentice =
the first *standing* holder). Assignments live on the Party
(`roleAssignments`, mirrored to the durable record; a departing member's
role is released with them; an assignment a formation switch orphans is
simply inert).

## The three hooks — exactly

1. **Target allocation** — `pickTarget` dispatches on the actor's
   resolved allocation: `sustain` (the verbatim pre-formation body — the
   byte-parity branch), `called` (Focus Fire: the captain's called
   target), `primary` (MA: the primary role-holder's own sustained
   target; the holder itself sustains). Every branch degrades to
   `sustain` when its premise is vacant. The **call is derived, not
   verb-called**: the captain leads by attacking — the side's called
   target is the captain's own entry engagement (`deliberateTarget`,
   stamped at open/join) while it lives, else their first sustained edge.
2. **Interception** — `runInterceptionPass` at the top of every beat
   (so an edge opened on beat N redirects at the top of beat N+1 —
   "lands next beat" by construction, and a player's own mid-beat
   `defend` interpose is never un-done within its beat): any threat edge
   onto a protected role whose trigger fires (`any`, or `high-threat` at
   `combat.formation.ma.highThreatEdges` incoming) redirects — via the
   session's own `CombatGraph.redirect`, never a content-facing surface —
   to the first eligible interceptor: interceptor roles in priority
   order, holders in roster order, standing, not the defender, under
   `combat.formation.intercept.maxIncoming` incoming edges. Deterministic
   throughout; witnessed (`narrateInterception`, a murmur-tier reactable
   beat).
3. **Coup governance** — the victor side's formation governs the
   two-stage death's stage 2 (see below).

## Coup governance: the right, the call, and the facts

The coup is engine-initiated (there is no coup verb — `beginCoup` fires
at incapacitation); governance re-points it:

- **The right** (`coupRight`): `engaged` keeps the downing attacker
  (byte-parity default); a role name re-points the stroke to that role's
  first standing, co-present holder — MA routes it to the *apprentice*,
  who performs the act and therefore **banks the deed** (credit routes by
  role structure; there is no credit transfer anywhere).
- **The call** (`coupCall: captain`): the coup does not auto-begin — it
  is **held** (the victor stands over the fallen "awaiting the word",
  `narrateCoupHeld`) for the captain's **`fight finish`** directive, and
  expires **spared** after
  `combat.formation.coup.directiveWindowSeconds` (mercy by default;
  `intervene` and walking away also still work). A captain-executioner
  needs no directive — the performer holds the call.
- **The facts**: the death row records `formationPath` / `killerRole` /
  `directedBy` (the ordering captain) — recorded fire-and-forget, never
  consulted by the crime rule. `BlameVerdict` grows the derived
  **`commandResponsible`**: a crime row carrying `directedBy` names the
  commander alongside the striker, so **credit and blame diverge** on an
  unlawful directed kill (the apprentice holds the deed, the master the
  command responsibility). Engine = facts; institutions = legitimacy.

## The `command` Discipline — teaching pays

Seeded at `seeds/lib/advancement/Discipline/command.yaml` (no
`conferrals` — competence grades, it doesn't gate). Minted (player-driven
actors only, the `mintExchangeSignature` parity) for **policy work
performed for someone else**: an interception taken, an armed opening an
ally cashed (`openingArmedBy` bookkeeping — the window itself stays
ownerless on the defender's poise, which is exactly the
ally-exploitability the MA economy stands on), the captain's target call
landing, the coup directive, the formation shift. The master/captain
advances the one discipline you cannot grind solo. **Master-Apprentice
has no reward knobs**: each actor banks their own signatures; the economy
holds because defense is generative, openings are ally-exploitable, and
competence is the exchange rate — the gym proves the mentored apprentice
survives a foe the same apprentice cannot beat alone.

## Surface

- **`party adopt <name>`** / **`party assign <role> <member>`** —
  captain-gated subcommands on the existing `party` verb (`adopt`, not
  `formation`, to avoid the `party form` adjacency). Adopting is a
  **witnessed, reactable formation-shift beat** ("the line reforms
  around…", topic `self.group`); a mid-fight adopt acts on the
  very next beat. `PartyLogic.setFormation` awaits the preset Idea
  resident before accepting (the beat's consult is sync);
  `AttackController` warms both sides' formations pre-open.
- **`fight`** (bare) shows your own side's `Formation:` + `Role:`; the
  enemy's formation is deliberately unread (the fog non-goal).
- **`fight finish`** — the captain's execution directive for a held coup.
- The **combatant brain** is formation-aware through the same chain a
  player resolves (NPC≈PC): a protector-role holder (front/master) leans
  into opening-creation (feints steady foes every beat — its windows feed
  the ward).

## Dials

| Key | Default | Meaning |
|---|---|---|
| `combat.formation.intercept.maxIncoming` | 2 | An interceptor already pressed by this many incoming edges is ineligible |
| `combat.formation.ma.highThreatEdges` | 2 | MA's `high-threat` trigger threshold (incoming edges on the protected role) |
| `combat.formation.coup.directiveWindowSeconds` | 12 | How long a captain-call coup is held before the fallen is spared |

## Proof

Unit suites (`CombatLogic.test`): per-preset allocation + interception
(including plural-role roster order and the ally-exploitability pin), the
mid-fight-adopt next-beat pin, coup governance end-to-end (held →
not-the-captain refused → the word → the apprentice delivers → the row
facts), mint spies, totality (a party of 1 on every preset), determinism.
The **gym formations matrix** (`scripts/__tests__/combat-gym.test.ts`)
proves the headline claims headless: default-formation byte-parity, Focus
Fire bursting the called target strictly faster than default, the back
line never falling under Vanguard, the MA emergent mentorship pair
(with-master survives / unassisted falls), and bit-for-bit cell
reproducibility. (The gym cannot witness signature minting — gym fighters
are brain-driven and mints deliberately skip brains — so the credit
claims are carried by unit-test spies.) The in-world demonstrator is the
newbie-wilds sellsword 2v1 (see the seed's comment header for the
script) — zero new NPC content.

## Deferred

- **Skirmish/Kite** — dies and revives with ranged combat (its own
  engagement-mode build).
- **Phalanx** (the mutual-cover protector variant) and any fifth preset —
  add when content asks.
- **Reading the enemy's formation** (a fog-hedged `assess` face).
- **An explicit call verb** (`party call <target>`) — the derived call
  covers v1; a clean later addition to the same read.
- **Shift costs** (poise/tempo on a mid-fight adopt) — wait for evidence
  of shift-spam.
- **The client `CombatPane` formation selector** — verb surface only
  until the pane lands.
- **Bang!-style composed rosters** (role decks scaled to party size) —
  a different feature; no preset wants it yet.

## Cross-references

- **Consumed:** [combat](./combat.md), [party](./party.md),
  [advancement](./advancement.md), [accountability](./accountability.md),
  [behavior](./behavior.md) (the `combatant` brain), [reactions](./reactions.md)
  (the shift + interception beats).
- **Design surface:** combat-tactics-slate,
  combat-slate.

## History

Built as the combat-formations build (branch `feature/combat-formations`,
`e05f0070..bc401ed9`, MR !148); requirements + plan retired at the
pre-merge sweep. Two design shifts happened between the seeding slate and
the build: the vocabulary moved from "tactic" to **formation** (with the
verb `party adopt`, dodging the `party form` adjacency), and the slate's
Master-Apprentice **reward knobs (scaled rewards / level-gap caps) were
superseded** by the emergent no-knob economy the combat experience pass
made possible. One as-built discovery reshaped the plan's coup section:
the coup is engine-initiated (no coup verb exists), so "who may coup"
landed as executioner re-pointing plus the `fight finish` captain-directive
hold rather than a permission gate on a verb.
