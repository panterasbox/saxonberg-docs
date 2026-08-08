# Combat

The **1v1 fight** — a witnessed, terms-bound session that runs a fast
tactical minigame (**poise**) at an **emergent tempo**, writes every
consequence through the shipped substrate ([vitals](./vitals.md),
[harm](./harm.md), [materials-response](./materials-response.md)), and
narrates each beat per-viewer. Combat is a **consumer**, not a new physics:
it computes no damage and stores nothing lasting on the `Creature`. It
picks a *mechanism* (a `Channel`), a *site*, and an *energy*, and routes
them through `ConditionApi.inflict`; the covering stack returns the trauma
type and severity. The one genuinely-new engine piece is **poise**; the
rest is a combat-specific arrangement of existing seams.

This doc is the source of truth for **cycle 1, builds 1 + 2** (the terminal
1v1 core, then its consequence/progression half). Build 2 adds the layer
that gives a fight lasting stakes: a **person** is only *defeated* by the
winning blow (killing them is the separate, interruptible **coup**), who is
to **blame** for a death is derived from an append-only ledger (a consented
duel is lawful; an imposed lethal attack on the unwilling is a crime), and
every exchange **advances** the fighter (combat `Discipline`s + a costed
`assess`). **Cycle 2** turns the 1v1 into an N-party melee — a threat
graph, join/merge, focus-fire, the `defend` family, fleeing, and the party
friend/foe seam (see [§ Cycle 2](#cycle-2--multi-party-the-threat-graph--the-party-seam)
+ [party.md](./party.md)); the client pane and NPC-vs-NPC crews are later.
Design surface lives in
combat-slate +
combat-experience-slate.

## The session — a `SustainedEngagement`, the dialogue twin

A **combat session** (`lib/combat/CombatSession.ts`) is a plain class
implementing `SustainedEngagement` — a near-exact structural clone of
`DialogueConversation` ([npc-dialogue](./npc-dialogue.md)). It is **not**
`Stuff`, never `StuffApi.create`d; it lives in the `SchedulerRegistry`
active set keyed to a combatant, exactly like `DialogueConversation` /
`RespirationDrain`. It holds the two combatants, their `CombatTerms`, each
side's transient `CombatantState`, the beat counter, and the resolution.

Two-sided, like a conversation: the session holds the `body` slot on
combatant A, and a companion `CombatPartnerHold` holds `body` on combatant
B (so both are genuinely occupied — movement is vetoed — and destroying or
aborting either tears the whole fight down through mutual-idempotent
cancel). The fight drives itself on a **fixed game-time cadence** via one
recurring `ScheduledEmission` (`combat.tickSeconds`, the `RespirationDrain`
precedent) — the emission fires inside `runRootGuarded(SchedulerApi,
'emission', …)`, so `tick()` has a well-defined root but **no** `commandId`
(the narration mints its own). Resolution/teardown routes through a single
`onAbort`; combat augments the `AbortReasonRegistry` (`combat-resolved`)
from its own module (the declaration-merge pattern).

The session is deliberately **thin**: it owns lifecycle + state and forwards
every *rule* (exchange resolution, poise mutation, narration, resolution)
to the gated `CombatApi`/`CombatLogic` pair. `tick()` is the only per-beat
entry; it calls `CombatApi.advance(this)`.

## Poise — the one new subsystem

`lib/combat/Poise.ts` — a fast, **session-scoped, banded** tactical gauge,
one per combatant, that evaporates at session end and **never touches the
`Creature`**. The model:

- Base autocombat **erodes** both sides each exchange.
- Committing a gambit **spends** the actor's own poise (overextend) —
  aggression has a price.
- Crossing the break floor arms a **binary, timed opening**
  (`{open, expiresAtTick}`) — the combatant reads `open` for a bounded
  number of ticks, then, unexploited, the window lapses.
- Defensive/reactive play (a **defend** beat) **restores** poise, capped by
  the `endurance` reserve ([metabolism](./metabolism.md) / `lib/reserve.ts`)
  — a gassed fighter can't buy it back.

**Bands, not numbers** across the whole surface: the raw scalar is private;
the only readout is `band()` (`steady | pressed | reeling | broken | open`)
and the binary `isOpen()`. Band thresholds + magnitudes are injected by the
session from `combat.*` AppSettings, so `Poise` stays a pure, tunable state
machine with no I/O (unit-tested standalone).

## Tempo — emergent cadence

`lib/combat/Tempo.ts` — **no attacks-per-round scalar**. Each combatant
accrues tempo at a *derived* rate and acts when the accumulator crosses a
whole exchange; the fractional remainder **carries**, so a faster fighter
simply acts more often (combat-slate #15). The rate is a pure product of
the three inputs that exist today — **encumbrance × endurance ×
competence** — times **weapon balance**, a neutral-default data hook
(`balanceFactor` on `lib/equipment/Weapon`) the deferred weapon-playstyle
cycle populates from construction without touching the function. Competence
is neutral in build 1 (combat `Discipline`s land in build 2); the seam is
live. The shape lives as one function so the load-bearing competence→cadence
curve is isolated and `combat.*`-tunable.

## Terms & consent

`lib/combat/CombatTerms.ts` — a value-object holding **lethality**
(non-lethal / lethal) × **stop-condition** (first-blood / yield /
incapacitation / death) × free-text **stakes** + the consent/initiation
record. Terms are consented like introductions: standing combat settings
pre-answer the handshake silently, and the pure `reconcile(mine, theirs)`
returns `agreed` (fold silently) or `conflict` (prompt) — conflict is
*only* a lethality mismatch (someone brings lethal to your non-lethal). A
frictionless bar scuffle opens with no prompt. `AttackController` runs the
reconcile, prompts a live defender's `Interactive` on conflict, and — when
the defender is an NPC with no session to consent — **imposes** the terms
`consented: false` (the crime marker build 2's blame ledger reads).

## The exchange — consequence through materials-response

`CombatLogic.commitInflict` builds an `InflictSpec {mechanism: Channel,
site, energy}` and calls `ConditionApi.inflict` ([harm](./harm.md)). Combat
picks the **channel** (from the resolved instrument — a wielded weapon's
delivery channel, or a species' innate attack; an innate now rides the
**same delivery split** as a weapon, so a `heat` innate builds a heat spec
and a **`shock` innate delivers through the instrument seam's drain** with
no mechanical primary at all — see
[combat-hooks.md](./combat-hooks.md) § the species vocabulary), the
**site** (torso by default; a called shot to the head on an open window),
and an **energy** derived from the target's **poise band** at the moment of
the blow (an open window earns the hardest hit — a `combat.energy.*` dial),
scaled by the **bounded instrument-delivery scale** (the crafting-branches
coupling): a weapon strike's energy is multiplied by
`MaterialApi.gradeConditionScale(grade, condition)` (the ONE quality
formula the covering stack already folds — Settled-4: quality scales
height, never shape), × the weapon's `keennessDeliveryFactor()` on
edge/point channels (the working-surface axis — see
[crafting.md](./crafting.md)), clamped down to
`crafting.brokenDeliveryFloor` when the weapon `isBroken()`. Material
*height* is deliberately left analyze-only (a combat-balance question —
the delivery fold reads quality, not hardness). It computes **no damage**;
`inflict` resolves the covering stack outside-in
and returns the trauma type + severity, or `null` (deflected). Armor
coverage measurably changes the outcome (a plated torso turns an edge a
bare one wouldn't) — that is entirely materials-response, not combat.

**Wear-on-use** (Law 2, the repair economy's producers): a landed weapon
strike wears its weapon (`crafting.wear.weaponPerStrike`) and dulls an
edge/point working surface (`crafting.keenness.wearPerUse`); on the
struck side, each covering layer that attenuates a mechanical blow wears
(`crafting.wear.armorPerBlow`, in `ConditionLogic`'s fold). `repair`
restores condition, `sharpen` restores keenness — two independent axes.

## Extension hooks — the combat-hooks grammar

Combat carries a declared **wizard-facing extension tier** (the
combat-hooks build): `@hook` seams across three surfaces — **instrument**
(`CombatReactiveMixin`, the one marker mixin the engine scans for:
`onWielded`/`onUnwielded`, the `augmentInflict` compute hook, the
strike/struck/parry/bypass witnesses), **participant** (`CombatantMixin`
no-op terminals — session entered, exchange resolved, band changed,
downed, defeated, defeated-foe, coup begun), and **venue** (optional
`CombatVenue` Location hooks — opened, blood drawn, resolved) — at fixed
points of the beat lifecycle, with consequences funneled through the
engine-drained `CombatHookContext` queue and the whole engine lint-locked
against per-dynamic carve-outs (`check-combat-dynamics`). The gated
**`CombatApi.influence`** bridge (stagger/expose/steady) is the external
state-instruction surface. The seams exist here; the **choreography, the
determinism contract, the enchantment-via-shadows boundary, and the
species combat vocabulary live in
[combat-hooks.md](./combat-hooks.md)** — read that before writing any
combat dynamic.

The outcome is **deterministic given information** (poker, not slots): the
exchange result is a function of poise + instruments, never a die roll. An
overextended actor **whiffs** (self-opens); a steady, armed defender
**parries** (and may **riposte** — see reactive dispatch); an open window is
**exploited** (the decisive break → the target is downed). The "crit" is an
*earned* opening, not a dice event.

## Gambits — affordances, attempt-time gated

`lib/combat/Gambit.ts` — the demonstrative set (`strike` / `disarm` /
`subdue` / `shove` + the reactive `riposte`) as **descriptors**, not a
bespoke registry: a gambit asserts a `{capability, band}` requirement
(channels-not-nouns) and whether it routes through `inflict`. Instruments =
body parts + a wielded weapon; a strike needs a **usable melee instrument**
(a wielded weapon in a functional, non-`disarmed` grip, or a
species-declared innate attack — `Species.naturalAttacks[]`, a
**multi-attack list rotated deterministically by beat**, with the legacy
single `Combatant.naturalAttackChannel` kept as a byte-preserving
fallback; the natural profile derives from the body, and a species can
afford existing gambits bodily — see
[combat-hooks.md](./combat-hooks.md) § the species vocabulary).

**Attempt-time cross-gating** (`CombatApi.eligibilityFor`): a gambit needing
an impaired slot is *rejected when attempted* (`Vitals.isSlotImpairedByTrauma`
→ the wielding grip's `bodyPart`), and `disarm` is rejected when the
opponent is unarmed. This is the "injury edits the menu" behaviour — the
visible menu-greying is a client-pane concern, deferred with the pane; the
terminal build needs only the reject. Humanoids declare **no** innate attack
(fisticuffs deferred), so a disarmed or fractured-grip humanoid genuinely
loses `strike`, while a natural-weapon beast keeps it.

`lib/combat/CombatFlags.ts` — the session-scoped flag set (`disarmed` /
`prone` / `grappled` / `inspired`), set by control gambits, gone at session
end.

**Reactive dispatch (X)** — the session consults a defender's reactive
affordances at `parried` / `whiff` / `grab`. "Reactive" is a net-new
notion (every `CommandDefinition` is actor-initiated); combat carries the
marker on the `Gambit` descriptor and filters eligibility. A parry fires the
defender's `riposte` (an offensive counter, weapon-only — a disarmed
defender can't riposte).

## Narration — the fight as an arc

`lib/combat/CombatNarration.ts` — the adapter that turns a resolved exchange
into **per-viewer** prose (Thesis 17). Per the plan's [VERIFIED-CORRECTION],
a single `Scene` fans by audience *bucket* and gives every peer the same
body (only per-viewer *naming* is automatic), so the adapter **loops the
room's witnesses and emits one `Scene` per perception tier**
(`MessageApi.scene(viewer).topic('act.combat').meta({commandId}).
toSelf(body).send()`, the `SocialLogic` presence-relay pattern) — combatants
read the precise band, bystanders a hedged clause. It mints its own
`commandId` (`SecurityApi.uuid()` — the beat runs in a detached scheduler
root) and calls `ReactionApi.noteReactableAct` itself on **dramatic beats
only** (`act.combat` is in `REACTABLE_TOPICS`; tick/pressed stay
silent).

The line is composed from the **tactical state** so the feed reads as a
**beginning → middle → end** arc, not a repeating template:

- It **escalates with the defender's poise band** — feeling-out (a composed
  guard) → pressure (`reeling under it`) → the break
  (`Its guard breaks wide — … through the gap`). The poise arc *is* the arc.
- Phrasing **rotates by beat** (channel verbs, wound words, parry lines) and
  verbs **conjugate per tier** (no "a wolf *land* a blow"); the defender's
  possessive is a **pronoun** for a combatant viewer (no name repetition).
- **State changes are surfaced**: an exploited opening reads as the break, a
  freshly-cracked guard is called out, control gambits name the flag.
- The **resolution names the cause** of the fall, read from the victim's
  worst wound (`bled white` / `skull broken` / `beaten past enduring`) —
  **every** resolution announces itself through one `endWith()` chokepoint,
  so a fight never just stops (the silent bleed-out gap, fixed).

The **flavor lookup** (`lib/combat/CombatFlavor.ts`) is a `{aspect, key,
channel, outcome}` table (`aspect ∈ material | species | gear | biome`) —
**NOT** fields on `Material` (which has a closed `fieldMeta`'s persistent entries list,
and one lookup serves material *and* species/gear/biome fragments layered by
generality). A miss is the common case and costs nothing — the frame is
always complete without a fragment (graceful default). Build 1 realises the
table as an authored code constant with the content-pack schema; migrating
it to a boot-warmed content catalogue is a follow-up (no consumer change).

## The enemy — a combat brain, invoked directly

`lib/behavior/combatant.ts` — the default enemy fighter (the `brain`
category). Unlike witness/cadence brains, the session invokes it **directly**
at its decision points by hand-building a minimal `BrainContext` and calling
`act` (bypassing `_runAct`'s slot-contention / presence machinery —
acceptable, the session owns its own concurrency). Asymmetric by design: the
full directed-autocombat loop is player-side; the enemy is brain-driven and
simply *picks a gambit* through `CombatApi.queueGambit`, holding fire when
overextended so the engine's defend-and-recover default takes over. A
non-player combatant (no live `Interactive`) is auto-detected as
brain-driven.

## Command surface — `attack` + `fight`

New command category **`combat`**. Two verbs (consolidated from the
initial seven per MR review — see [§ History](#history)):

- **`attack <target> [--lethal] [--to <stop>]`** (`AttackController`) —
  opens a fight; runs the terms handshake.
- **`fight`** (`FightController`, dispatch-on-subcommand, the `bank`/`chat`
  precedent) — everything *inside* a fight: bare `fight` (or `fight status`)
  is the at-a-glance read (own poise/condition/flags/wounds at full fidelity
  + the banded, perception-gated opponent read — all bands, never numbers);
  `fight strike | disarm | subdue | shove` are the gambits (queued intent,
  resolved on the next beat); `fight defend` covers up and recovers poise
  (capped by endurance) instead of attacking — the autocombat default
  handed to the player as a deliberate choice, so they can play the
  patient defender on purpose (the build-1 balance finding: a steady armed
  defender's parry-and-riposte beats a blind aggressor); `fight yield`
  concedes.

Both verbs are contributed by **`CombatantMixin`** (`lib/combat/Combatant.ts`,
composed on `Character`) as static `self` affordances; the gambit
subcommands reject at attempt-time. `CombatantMixin` also carries the
`naturalAttackChannel` field (now the **legacy single-attack fallback**
under `Species.naturalAttacks[]` — see
[combat-hooks.md](./combat-hooks.md)) and a banded,
perception-safe `look` augmenter (the combat-state line while a fight is
live). The `wield`/`unwield` affordance — shipped as verbs with the
"Weapon is holdable" build but deferred at the affordance layer — is wired
here on `WieldableMixin` (arming yourself is a combat prerequisite;
[embodiment](./embodiment.md)).

## The demonstrator — the treeline cull

`attack` a brain-driven **wolf** (`/obj/species/wolf` quadruped +
`/domain/newbie-wilds/npc/wolf`, a `wolfshead` bounty) in **the treeline**
(`/domain/newbie-wilds/crossroads/treeline`, off the crossroads hub, reached
via the char-gen `startLocation` override, no inbound exit wired — content-
area standup stays clean). Pick up + wield the dropped steel dagger, `attack
wolf --lethal`, and the poise/gambit fight runs to a lethal resolution
(non-sentient → no consent, no blame; the cull). This is the "reachable in a
demonstrator context" acceptance, live in shipped content.

## `CombatApi` / `CombatLogic`

The gated pair (`api/combat.ts` + `obj/api/CombatLogic.ts` at
`/obj/api/combat`, `FromModule('/api/combat#CombatApi')`) is the **sole
entry** to lifecycle / gambit-resolution / poise-mutation / advance /
narration, mirroring `ConditionApi ↔ ConditionLogic`. `CombatLogic`'s class
methods are thin gated entry points; the rules engine lives in
module-private functions (the `ConditionLogic` precedent — nothing routes
through the instance proxy mid-algorithm). All tuning is `combat.*`
AppSettings seeded from `config/app-settings.yaml` (no code defaults):
tick, poise thresholds/costs, tempo shape, per-band inflict energy,
max-beats.

## Build 2 — consequence & progression

Build 1 makes a fight *happen*; build 2 makes it *matter*. The exchange
engine is unchanged — build 2 hangs consequence off the resolution seams.

### `isSentient` — the person / beast axis

`SpeciesApi.isSentient(o)` (the `isAnimate` twin) reads a `sentient` flag on
`Species` (default false; the whole `homo` genus + the tutor-bot construct
are `sentient: true`, wolf / frog / plant are not). It is the key to the
**three-case** severity of a defeat (`CombatLogic.handleDown`):

- **non-sentient + lethal** → the **cull**: the winning blow finishes the
  beast (stage 2 skipped), no consent, no blame (build 1, unchanged);
- **sentient + lethal** → the fight resolves at `incapacitation` (the poise
  contest only *defeats*); the kill is the separate **coup** (below);
- **sentient + non-lethal** (or an unauthorized finish) → `incapacitation`,
  nobody dies.

### Two-stage death — the coup

`lib/combat/Coup.ts` — a `DurativeActivity` (the `ManualBuildStep`
precedent) that holds the victor's `body` for `combat.coupSeconds`
game-time. Killing a downed **person** is not the winning blow; it is a
slow, telegraphed, **interruptible** killing stroke begun *after* the fight
resolves at incapacitation (deferred one tick via `ScheduleApi.schedule(0)`
so the session frees the `body` slot first). On completion the kill lands
(`setLifecycleState('dead')` + a `causeOfDeath` stamp + the death
attribution row + the resolution consumers); on abort the victim is spared.
`CombatNarration` gives it its own **telegraph** ("stands over the fallen …
the stroke poised") and **stayed** lines. The new `combat-intervened` abort
reason is declaration-merged from `Coup.ts` (the `combat-resolved`
precedent). `coupEligible` re-checks co-presence + not-already-dead at both
start and completion, so dragging the victim clear also spares them.

The coup follows **incapacitation, however it was reached** — both the
poise-contest down (`handleDown`) *and* a bleed-to-unconscious by attrition
(`checkVitalsResolution`) begin it for a sentient loser under lethal terms.
(The attrition path was the common outcome in the live demo — the
defender-advantage from build 1 means a straight fight usually ends by
accumulated wounds rather than a clean poise break — so wiring the coup to
attrition too is what makes the two-stage death reliably *reachable*.)

**`intervene <target> | stay`** (`combat/intervene.yaml` +
`InterveneController`, afforded by `CombatantMixin`) cancels a live coup in
the room — matched by **victim or executioner**, so a bystander can stop the
killer, a friend can shield the fallen, and the victor can stay their *own*
hand (mercy). This is the moral crux: not-killing is a move you can play.

### The blame ledger (derive-on-read) — now the shared accountability ledger

> **History (stealth & deployables build).** This ledger was **extracted
> out of combat** into a harm-agnostic substrate — `lib/accountability/`,
> collection renamed `combat_attribution_events` → `accountability_events`,
> `CombatAttributionEvent` → `AccountabilityEvent` — so ambush, sprung
> traps, and combat all feed *one* crime derivation. Combat migrated onto
> it **byte-identically** (a pinned regression pins every combat verdict
> unchanged) and is now the ledger's first *consumer*, not its owner. The
> `death`-row rule below is verbatim; the substrate adds a `harm` kind for
> non-combat producers. See [accountability.md](./accountability.md).

An append-only `Document` (the `RenownEvent`/`AuthoringEvent` shape; durable
ids are `templatePath`s). Combat never stamps a "murderer" flag; it records
the objective facts and **derives** culpability on read. Combat's three
writers, no others: `opened` (initiator + terms + consent + sentience, at
`openSession`), `violated` (the standalone crime marker, written at
initiation when lethal terms are *imposed* on a non-consenting sentient),
and `death` (victim + killer + the terms in force) — each now delegating to
`AccountabilityApi.record`. The pure reader `deriveBlame(rows)` takes the
**earliest** terminal row (the `ProvenanceApi.authorOf` rule) and returns
the killer plus `crime` — for a `death` row, **true iff a sentient was
killed under lethal terms they did not consent to**. Surfaced by
`CombatApi.blameFor(victimId)` / `attributionFor(sessionId)`, both thin
delegations to `AccountabilityApi`. Re-legislating what counts as a crime
re-scores history without rewriting a row. Writers are fire-and-forget (the
ledger never blocks a beat).

### Consent posture — the handshake made real

Build 1's `AttackController` already read `combat.lethality` /
`combat.stopCondition` standing settings, but **no schema declared them** —
so no one could hold a lethal standing and a consented *lethal* duel was
impossible (every lethal attack was imposed → crime). Build 2 makes the
posture readable from **two surfaces**, because the two audiences carry it
differently:

- **players** — the `combat.lethality` / `combat.stopCondition` **settings
  schema** on `CombatantMixin` (`Enum`, defaults `non-lethal`/`yield`),
  which a player sets at runtime (`settings set combat.lethality lethal`).
  Settings only resolve for `Environment` hosts.
- **NPCs** — the authored `CombatantMixin` **fields** `standingLethality` /
  `standingStopCondition`. An NPC **isn't an `Environment`**, so it can't
  carry a setting override — a duelist authors its posture in these fields
  instead (`standingLethality: lethal`).

`standingTerms` reads both: either surface declaring `lethal` makes the
combatant bring lethal terms. So the gentleman duelist's authored
`lethal` posture **matches** a `--lethal` challenge → the terms reconcile
silently to a **consented** duel (no prompt, no crime). The same stroke
against a townsperson at the non-lethal default is imposed, and the
`violated` marker fires. *(This NPC-can't-hold-a-setting gap was found and
fixed during the live demo — the settings-only approach the plan assumed
never reached NPCs.)*

### Resolution consumers

At every win / kill, `runResolutionConsumers` (defensive, best-effort): a
`ChronicleApi.recordDeed` for the victor (a **deed**, or a **crime**-tagged
line for an unlawful kill) + a `RegardApi.adjustRegard` nudge from every
room witness (`combat.regard.duelWin` for a clean win, the negative
`combat.regard.unlawfulKill` when the room recoils from a murderer). The
global "X killed Y" presence relay stays deferred — the room-scoped death
narration already announces it.

### Advancement + `assess`

Two combat `Discipline`s are seeded as data (the `bartending` precedent,
auto-harvested by `DisciplineCatalogue`): `melee-combat` (skill) and
`blades` (specializes it). Each resolved exchange mints the **player
side's** own `ActSignature` (self-credit; difficulty from the target's
poise band, outcome from the exchange result; `blades` additionally
credited on an edge/point instrument) via `AdvancementApi.recordSignature`,
fire-and-forget. The existing **`assess`** verb (`AssessController`) gains a
mid-fight branch: assessing your opponent delegates to `CombatApi.assess` —
a **costed** read (it spends your next exchange, `queuedGambit = 'assess'`)
that mints a combat signature and returns the opponent's banded tactical
state (poise / flags / armed / condition). Bands, never numbers.

### The demonstrator — the consented duel

`attack` **the gentleman out of the fog who works with a knife** (the
crossroads bounty the board already names): `/domain/newbie-wilds/npc/
duelist` — a sentient human whose standing terms are lethal, waiting in the
**fog hollow** (`/domain/newbie-wilds/crossroads/hollow`, one step west of
the treeline). He takes up a knife from the duelling-stone via the new
`arms` brain (`lib/behavior/arms.ts` — wield the nearest reachable weapon;
no NPC-arming engine change). `attack the gentleman --lethal` → the terms
agree silently (a consented duel) → win the poise fight → he is **defeated,
not killed** → the coup **telegraphs** → you let it land (a *lawful* kill —
chronicle deed, a small regard cost) **or** `intervene` to spare him. The
**crime** leg is reachable without extra content: `attack` any
non-consenting sentient (a townsperson) `--lethal` → the `violated` marker →
`blameFor` returns `crime: true`.

## Cycle 2 — multi-party (the threat graph + the party seam)

The 1v1 core goes plural. The fight becomes an **N-participant melee** and
reads friend-from-foe off the party operational core through one narrow
seam (see [party.md](./party.md)).

- **The session is now a plain N-container**, not a `SustainedEngagement`.
  `CombatSession` holds a `Map<Stuff, CombatantState>`, a `CombatGraph`,
  and its own **real-time** beat via `ScheduleApi.recurring` (the beat no
  longer rides any participant's game-time emission — so a participant
  leaving never orphans the tick, and the cadence is pace-independent of
  world-clock scale, the coupling the 1v1 live demo exposed). **Bleed stays
  game-time** (harm reconcile-on-read untouched). Every participant carries
  one **uniform `CombatParticipantHold`** on `body`; a departure removes
  just that participant, dissolving only when a side empties. `sessionFor`
  is a uniform participant-hold lookup.
- **The threat graph** (`lib/combat/CombatGraph.ts`, pure/unit-tested) is a
  directed who-attacks-whom graph. Each `ThreatEdge` carries its **own**
  `CombatTerms` — a duel and an interloper's unlawful blow in the *same*
  session carry different terms (the per-edge blame foundation). Death /
  blame sites read `termsFor(session, killer, victim)` (the killing edge's
  terms, falling back to the session's for the degenerate 1v1);
  `CombatantState.lastStruckBy` names the killing edge for an attrition
  death.
- **Sides + join / merge.** Each participant's `PartyApi.sideOf` is frozen
  on its node at open/join; the exchange loop is **side-driven**
  (`pickTarget`: a foe is anyone not on the actor's side). `CombatApi.join`
  adds a participant + `body` hold + a mutual edge; `CombatApi.merge` folds
  two colliding fights (holds reparent onto the survivor). `AttackController`
  is the handshake: attack someone already fighting → join; attack from
  mid-fight → draw the target in; two fights collide → merge; else open.
- **Focus-fire** (the balance fix / turtle-breaker): a target's poise
  erosion scales with `graph.edgeCount(target)` (each attacker beyond the
  first adds `combat.focusFire.erosionPerEdge`), and a defender pressed by
  ≥ `combat.focusFire.suppressRecoveryAt` attackers can't spend a beat
  recovering — the lone turtle beats one but loses to two.
- **The `defend` family** — one canonical `defend` verb over three
  cardinalities: `defend` (cover up = the self gambit), `defend <fallen>`
  (stay a coup = the old `intervene`/`stay`, kept as aliases), `defend
  <ally>` (interpose — `CombatApi.defendAlly` joins if needed and
  `CombatGraph.redirect`s a foe's edge off the ally onto the interposer).
- **Fleeing** — combat's resolution of a **locomotion attempt made while
  engaged** (not a verb, not a mode). `CombatApi.disengage`, called at the
  movement controller's pre-traverse gate, is a no-op when not fighting
  else an **opposed-lite** break: a focus-fire pin vetoes it, foes still
  locked on get a **parting shot** (same materials-response inflict, at
  `combat.flee.partingShotEnergy`), success removes the actor and the
  traverse proceeds. Individual only.
- **The NPC ally** — the `combatant` brain is side-aware (acts only while a
  live foe remains; targeting stays the engine's job). A new **`backs-up`**
  brain gets a party NPC *into* a fight: it watches for a party-mate who
  has drawn steel and `CombatApi.join`s on their side. The **demonstrator**
  is a recruitable `Mercenary` sellsword at the treeline: `party form`,
  `party enlist the sellsword`, walk west, `attack the gentleman --lethal`
  → she backs you up for a consented **2v1** in the hollow.

The reachable multi-party demonstrator is the 2v1 above; an NPC-vs-NPC
**crew** (two NPCs sharing a side without a live player forming the party)
needs a durable-party standup/seeder and is deferred. **Split** (a fight
fragmenting into two sessions on edge removal) has no acceptance criterion
and is deferred until a demo produces genuine fragmentation.

## The experience pass — poker, not slots (the feint, the fog, the gym)

Builds 1+2 make a fight *happen* and *matter*, but the exchange had a dead
seam the live demos exposed: **a steady armed defender always parries, so
blind patience beat aggression** and there was no read-and-react tension.
The experience pass makes the exchange **poker, not slots** — the outcome is
determined by inputs you can't fully *see* (you lose because you *misread*,
never because a die spiked) — closes rock-paper-scissors so patience is no
longer a free win, and ships the tool that *proves* the balance.

**The feint** (`Gambit.ts` `feint`, the read/deception channel). A feint
inflicts nothing; it presents as bait and reads the *defender's*
commitment. A **committed** defender — a steady, armed turtle poised to
parry, exactly who patience-as-dominant-strategy relies on — who fails to
**read** the bait over-commits and cracks their own guard (`feint-bit`):
the defender loses `combat.poise.feintBitPenalty` (enough to break a steady
guard, arming their opening via the existing `Poise.lower` crossing), the
feinter pays only the small `combat.poise.feintCost`, and the opening is
cashed on the feinter's **next strike** through the unchanged `exploit`
path (the two-beat feint→exploit, no new inflict site). A defender who
**reads** it (competence-gated) or isn't committed (an aggressor, not
turtling) isn't fooled (`feint-read`, the bait wasted). This closes
**strike ▸ feint ▸ defend ▸ strike**: a feint beats a turtle, patience
beats blind aggression, aggression beats a feinter.

**The fog** (`lib/combat/CombatFog.ts`, pure, deterministic). The read is
*epistemic* — you see the opponent's poise **fogged by your own sharpness**.
`CombatFog.perceive(trueBand, sharpness, feinting)` distorts deterministically
(no RNG): a dull reader under-reads (band shifts one step toward `steady`)
and **mistakes a feint for a real opening** (`open`); a sharp reader sees the
true band and the feint's **tell**. Two surfaces carry the fogged read:
the **free** `fight` status opponent line (`CombatApi.perceive` — no cost,
no side-effects), and the **costed** `assess` verb (`CombatApi.assess`
wraps `perceive` and adds the beat cost + the melee-read `ActSignature`).
The **same** `CombatFog.reads(sharpness)` gate decides both the fog's tell
and whether a defender bites a feint, so "the fog says open" and "I bit the
feint" can never contradict. **The fog is the only dice; skill shrinks it.**

**Sharpness** (`lib/combat/Sharpness.ts`, pure). One scalar,
`f(competence) · g(composure)`, modulating **both** poise recovery (a
sharper fighter recovers more per defensive beat) and the read-fog.
`g(composure)` is the **inert** `traits-stress` seam (`≡ 1` today — a
stubbed composure composes without touching the exchange engine). Competence
is snapshotted **synchronously at open** (`CombatOpenOptions.competenceBands`,
keyed on `templatePath`; `AttackController` awaits `AdvancementApi.bandFor`
*before* opening — the async band can't be read mid-beat, and reading it once
keeps a single session deterministic). Bare/test/gym/NPC paths default to
`untrained`.

**Zero new aleatory randomness.** No to-hit / damage / crit dice; severity
stays the deterministic materials-response function, a "miss" stays a
*caused* event, the crit stays the **earned** opening (with the called shot
to the exposed part). **A single session is bit-for-bit deterministic** —
the property the gym rides.

**The dramatic arc** (Thesis 2). Every beat carries a
`BeatIntensity` — `roar` at a threshold crossing (first-blood /
the break, cracked or exploited / the down or kill — `CombatSession.bloodDrawn`
fires the first-blood roar once), `murmur` for an ordinary hit, `silent`
otherwise. It **volume-gates the crowd**: `CombatNarration` marks an act
reactable (`noteReactableAct`) only on murmur/roar, and scales the narration
swell — "the crowd roars" *is* the engagement feedback that the beat
mattered. (The reactions substrate carries no per-act weight, so intensity
gates *whether* the reactable act fires + the prose swell, not a counter;
an empty-room roar is the swell, and an automated bystander crowd is
deferred.)

**The combat gym** (`scripts/combat-gym.ts` + `scripts/__tests__/combat-gym.test.ts`).
Because a session is a deterministic single-thread coroutine, a whole fight
runs **headless, to resolution, with no scheduler** — loop
`CombatApi.advance`. A **dev tool** (no Api, no verb, no world state; the
`scripts/` precedent) that runs a matrix of skill × policy × loadout
matchups and reads the outcome distribution. It is the **balance regression
guard**: it asserts the parry seam is dead (a feint beats a low-competence
turtle; blind aggression still loses to one; an expert *reads* the feint and
isn't cracked), a contested win-rate (no policy sweeps), single-session
determinism, and **NPC ≈ PC** parity (the same model fights itself to a
decisive result). Variation is the matrix, never RNG. This de-risks the
competence→exchange-rate curve (combat-slate OQ1 — "numbers are the risk";
the gym is how you find them).

## Weapon playstyle & the hand-slot economy

A weapon is a **derived playstyle bundle**, never a stat block. From its
*shape* — `Construction` (form) × `Material` × `mass` × `length` × the
hand-slot claim count — a pure {@link WeaponProfile} (`lib/combat/`) computes
five axes, each an **input to a system that already exists** (no parallel
weapon engine):

- **balance** ← `mass`. Light = an **exploiter** (fast tempo, cheap commit —
  it *cashes* openings); heavy = a **guard-breaker** (slow, high poise damage,
  expensive commit — it *creates* them). The numeric getters
  (`tempoFactor`/`poiseDamageFactor`/`overextendFactor`) are what
  `CombatLogic` multiplies into `Tempo.rateFor` (the seam the old inert
  `balanceFactor` occupied — retargeted, the field kept only so old rows
  hydrate) and the exchange's erosion / inflict energy.
- **reach** ← `length` (a real magnitude, symmetric with `mass`). A banded
  class (`short`/`medium`/`long`) drives a **geometry-free engagement-range
  tier** on `CombatGraph`: each pair carries a band from the
  `close · reach · near · far` ladder
  (`ThreatEdge.range`, `setRange`/`rangeBetween` keeping both directed edges
  in sync). A longer weapon **controls until closed** — a genuinely out-ranged
  attacker (a 2-rank gap) is simply **out of range** and *whiffs* (offence AND
  riposte); reach-advantaged actors resolve first (a stable reach-order sort).
  ⚠ The ranged build widened that tier to four bands and moved the
  OPENING band off reach rank onto the room's real size — see
  [ranged.md](./ranged.md). Reach still decides who controls the gap;
  it no longer decides where the fight starts, and `close` is now an
  alias for `advance`.
  **Closing** is a tempo-costed opposed beat (the `close` gambit), *contested*
  by a composed, longer reach-holder; a reach-holder's `defend` re-opens
  distance. Inside (`close`) the term **reverses** — the dagger/unarmed owns
  the clinch, the spear is a liability. Per-edge, so a 2v1 carries mixed ranges.
- **guard** ← form (× a light material term). A crossguard parries + ripostes;
  the two **guardless** forms (`guard: none`, new weapon-delivery forms) can't
  self-guard, so a guard-breaker **bypasses** their steady defence: `flail`
  (blunt, a chained head) and `whip` (a cutting-lash `edge`, and the **long-
  reach extreme** — it controls at `reach` and is helpless inside). Rides the
  existing steady-guard → parry → riposte seam (the binary `targetCanParry`
  became a guard-graded read).
- **handedness** ← the `slotClaims` count (a 2H weapon claims both grips).
- **delivery** ← the `Construction` delivery form (unchanged).

Surfaced as **bands/pips** (`analyze weapon`, a combat-owned `analyze`
subcommand + a playstyle-pips `markupAugmenter` on `Weapon`); an
`check-inert-weapon` CI lint (the `check-does-nothing` sibling) guards that no
seeded weapon derives an inert profile.

**Shield = wielded armor-construction** (no `ShieldMixin`): a `Wieldable`
carrying an *armor* `Construction`. `ConditionApi.inflict` folds it into the
materials-response covering stack as **directional** front cover (the
`shieldFacing` hint — strong facing one foe, bypassed by a flank under
focus-fire keyed on `graph.incomingEdges`), it grants its holder a large
**guard** bonus (`combat.offhand.guardBonus`), it **costs a hand**, and it is
**sunderable** (`DurableMixin` — a worn-out shield turns less).

**The hand-slot economy** — two hands as a live allocation:
- **Switch** (`fight switch <weapon>`) — a vulnerable durative beat
  (`lib/combat/WeaponSwitch.ts`, a beat-driven countdown, NOT a scheduler
  activity, since the participant already holds `body`): guard down, no
  instrument, for `combat.switch.seconds`, then the grip swaps + tempo
  re-derives.
- **Sidearm draw** (`fight draw`) — a *fast* switch to a sheathed (`sidearm`
  slot) / carried backup: **disarm becomes a tempo setback, not a fight-ender**
  (the `combatant` brain draws when disarmed).
- **Dual-wield** — two weapons: the off-hand adds a guard bonus **band-gated**
  on competence (a novice fumbles it — a net penalty; you grow into it, the
  experience-pass competence seam).

**Weapon-shaped gambits** ("the weapon edits the menu" beside "injury edits
the menu"): `bash` (shield-afforded), `sweep` (hafted-afforded), and
`entangle` (whip-afforded — wraps the lash to bind a foe `grappled`; the
whip's real weapon, since it fights by *control* not the sting of the lash)
carry equipment requirements (`affordedByForm`/`affordedByShield` on
`GambitSpec`), gated at `eligibilityImpl` like any gambit; `analyze weapon`
lists a form's afforded moves.

**Determinism** is preserved — every axis is a deterministic function of the
tactical state (no new RNG); the **gym** gains a weapon × allocation matrix
(the `GymLoadout` axis) asserting no loadout strictly dominates + single-
session reproducibility with weapons (each matrix cell runs in a reset clean
world — a session is deterministic, but a matrix must not accumulate the prior
fight's objects). All tunables are `combat.weapon.*` / `combat.reach.*` /
`combat.switch.*`/`combat.draw.*`/`combat.offhand.*`/`combat.dualWield.*`
AppSettings. Content: the `steel-spear`/`steel-warhammer`/`steel-flail`/
`steel-sword`/`steel-shield`/`leather-whip` arms (in `/obj/arms/`) + the
`sidearm` biped slot.

## Deferred

Named at their sites; nothing inherited:

- **Later cycles** — ranged & thrown weapons (a whole engagement mode of its
  own), the deep grapple/choke control game, spatial formation/geometry, a
  weapon crafting/repair economy, full morale / de-escalation, stealth,
  pursuit / the chase (wayfaring) and coordinated party-retreat (rout/rally),
  non-humanoid bestiary, death/recovery + moderation, and the client
  `CombatPane` (and the contextual gambit affordances — terse verbs afforded
  only in a fight — that supersede the static ones). Party formation-roles
  **landed** — the combat-formations build (see
  [combat-formations.md](./combat-formations.md)): `CombatFormation` Idea
  presets over a total party-face resolution chain, the three hooks
  (allocation / interception / coup governance), the `command` Discipline,
  and the `party adopt` / `fight finish` surface.
- **The composure/luck axis itself** (`traits-stress`) — the bidirectional
  stress ↔ inspiration emotional reserve that fills `Sharpness`'s
  `g(composure)`. The experience pass leaves only the inert modulation seam;
  the axis, its accrual pipe, and the estimator are deferred.
- **A broader deception menu** — false-opening, guard-baiting, bluff-stacking.
  The feint is the one read gambit this pass; the wider mind-game is deferred.
- **Split** — connected-component recompute on edge removal spawning a
  second session; deferred (no criterion requires it) until fragmentation
  is real.
- **Known engine seams** — the `act.combat` topic wants a client
  font-register mapping. The beat loop has **no presence-freeze** — a fight
  ticks on against a linkdead combatant until `combat.maxBeats` forces a
  draw (bounded, not a leak; "you can't rage-quit a fight" may even be
  wanted). A parting-shot **narration** (the inflict is mechanical today) is
  a small polish. An **automated bystander crowd** (so an empty-room duel
  still draws a reacting audience) is a behavior concern, out of scope.

## Cross-references

- **Substrate consumed:** [vitals](./vitals.md), [harm](./harm.md),
  [materials-response](./materials-response.md), [activity](./activity.md),
  [reactions](./reactions.md), [metabolism](./metabolism.md),
  [behavior](./behavior.md), [belief](./belief.md),
  [messaging](./messaging.md), [prose](./prose.md).
- **Structural precedent:** [npc-dialogue](./npc-dialogue.md) (the
  `DialogueConversation` `SustainedEngagement` twin).
- **Design surface:** combat-slate,
  combat-experience-slate,
  combat-tactics-slate,
  party-slate.

## History

- **Cycle 1, build 1** (`feature/combat-core`, MR !131) — the terminal 1v1
  core: the session/hold/tick, `Poise`/`Tempo`/`CombatTerms`/`CombatFlags`/
  `Gambit`, the exchange-through-`inflict`, gambits-as-affordances +
  attempt-time gating + reactive dispatch, the arc-driven per-viewer
  narration + flavor lookup, the `combatant` brain, `CombatApi`/`CombatLogic`,
  the `attack` + `fight` command surface, and the treeline cull demonstrator.
  Incidental fixes forced by the live demo: the `wield`/`unwield` affordance
  (`WieldableMixin`), the missing `/obj/ParcelRegistry` seed (a latent
  fresh-DB bootstrap failure), the silent-fight-end bug, and the flat
  narration. `CombatantMixin` was composed onto `Character`; `Weapon` gained
  `balanceFactor`; `act.combat` joined `REACTABLE_TOPICS`.
- **Cycle 1, build 2** (`feature/combat-consequence`) — consequence &
  progression: `isSentient` (a `Species` flag + the three-case defeat),
  the two-stage **coup** (`Coup` `DurativeActivity` + the `intervene`
  verb + `combat-intervened`), the derive-on-read **blame ledger**
  (`CombatAttributionEvent` in `combat_attribution_events` +
  `deriveBlame` + `CombatApi.blameFor`), the real **consent handshake**
  (the `combat.lethality`/`combat.stopCondition` settings schema build 1
  read but never declared), the resolution consumers (chronicle
  deed/crime + regard), combat `Discipline`s (`melee-combat`/`blades`) +
  per-exchange `ActSignature` + the costed combat `assess`, the
  `fight defend` gambit, and the **consented-duel** demonstrator (the
  gentleman duelist + the fog hollow + the `arms` brain). Three new
  `combat.*` dials (`coupSeconds`, `regard.duelWin`, `regard.unlawfulKill`).
  **Two fixes forced by the live demo:** the coup was wired to fire on a
  bleed-to-unconscious incapacitation too (not only a poise-contest down),
  since the defender-advantage means fights usually end by attrition; and
  NPC consent moved from a settings-only approach (settings only resolve
  for `Environment`s — which NPCs aren't) to authored
  `standingLethality`/`standingStopCondition` fields on `CombatantMixin`.
- **Cycle 2** (`feature/multi-party-combat`) — the 1v1 goes plural (see
  [§ Cycle 2](#cycle-2--multi-party-the-threat-graph--the-party-seam)):
  `CombatSession` rewritten from a `SustainedEngagement` to a plain
  N-container with a real-time `ScheduleApi.recurring` tick + uniform
  `CombatParticipantHold`s; the `CombatGraph` value-object (per-edge terms
  + `lastStruckBy` attribution); side-driven targeting frozen from the new
  **party** seam (`PartyApi.sideOf`/`areAllied` — see [party.md](./party.md));
  `join`/`merge`; focus-fire poise economy (3 `combat.focusFire.*`/`flee.*`
  dials); the canonical `defend` family (`intervene`/`fight defend` become
  aliases; `defend <ally>` via `CombatGraph.redirect`); fleeing as an
  opposed-lite disengage at the movement controller's pre-traverse gate;
  the side-aware `combatant` brain + the `backs-up` brain + the 2v1
  Mercenary-sellsword demonstrator. The whole **party operational core**
  (`lib/party/`, `PartyApi`/`PartyLogic`, `PartyRegistry`, the `parties`
  collection, the `party:` grouping provider, the `party` verb) landed in
  the same build.
- **Experience pass** (`feature/combat-experience`) — the exchange becomes
  poker, not slots (see [§ The experience pass](#the-experience-pass--poker-not-slots-the-feint-the-fog-the-gym)):
  the **feint** gambit (`feint-bit`/`feint-read`, the two-beat feint→exploit
  reusing the earned-opening path) closing rock-paper-scissors and killing
  the always-parries seam; the deterministic competence-hedged **fog**
  (`CombatFog`) fed into `assess`; the `f(competence)·g(composure)`
  **`Sharpness`** scalar modulating recovery + fog (composure the inert
  `traits-stress` seam) snapshotted at open via `CombatOpenOptions`; the
  **beat-intensity** arc (`BeatIntensity` silent/murmur/roar +
  `CombatSession.bloodDrawn`) volume-gating the crowd; and the **combat
  gym** (`scripts/combat-gym.ts` + bench) proving the parry seam dead +
  single-session determinism + NPC≈PC parity. Six new `combat.*` dials
  (`poise.feintCost`/`poise.feintBitPenalty`/`fog.readSharpness`/
  `fog.clearSharpness`/`sharpness.min`/`sharpness.max`). `Gambit` grew a
  `feint` kind; `fight feint` joined the gambit surface. **Two fixes forced
  by the live run:** the fog was only on the costed `assess` path, so the
  free `fight` status still showed the raw opponent band — split a free
  `CombatApi.perceive` out of `assess` and used it in the status read; and
  `assess` had never been added to `PerceiverMixin`'s affordances (it was
  categorized as a perception verb, its AssessController branches wired, but
  it dispatched "I don't understand" for every character) — afforded it
  alongside `look`/`scry`/`feel`, and fixed a "They looks unhurt" number
  disagreement in its combat-read prose.
- **Weapon playstyle & the hand-slot economy** (`feature/weapon-playstyle`,
  MR !140) — a weapon becomes a **derived playstyle bundle** (see [§ Weapon
  playstyle & the hand-slot economy](#weapon-playstyle--the-hand-slot-economy)):
  `WeaponProfile` (reach/balance/guard/handedness/delivery from form × material
  × `mass` × `length` × hand-claims), the geometry-free `reach|close` tier on
  `CombatGraph`, shield-as-wielded-armor in the covering stack, the switch/
  sidearm/dual-wield hand-slot economy, and the `bash`/`sweep`/`entangle`
  weapon-shaped gambits. `analyze weapon` + a `check-inert-weapon` lint; the
  gym gained a weapon × allocation matrix; 23 new `combat.weapon.*`/`reach.*`/
  `switch.*`/`draw.*`/`offhand.*`/`dualWield.*` dials. Two **design→impl
  shifts** worth recording: (1) `WeaponSwitch` is a **beat-driven
  value-object, not a scheduler `DurativeActivity`** as the plan proposed — a
  combat participant already holds the whole `body` engagement slot, so a
  second `body`-claiming activity would conflict; the vulnerable window
  (`resolveInstrument` returns null for N beats) is unchanged. (2) The
  reach **out-of-range whiff** is gated on a genuine 2-rank gap
  (`REACH_OUT_OF_RANGE_GAP`), so a 1-rank gap (sword-vs-dagger, weapon-vs-
  natural) stays a mild energy nudge and the wolf/cull fights are unperturbed.
  A follow-on added the `whip` delivery form (guardless, long-reach extreme)
  + its `entangle` gambit. The generic arms/armor/gear/clothes/items seeds
  were relocated out of `domain/eternal/` into `/obj/` in the same branch.
- **Combat hooks** (`feature/combat-hooks`) — the extension grammar (see
  [§ Extension hooks](#extension-hooks--the-combat-hooks-grammar) and
  [combat-hooks.md](./combat-hooks.md)): `CombatReactiveMixin` +
  `CombatHookContext` + the `CombatVenue` interface; every engine dispatch
  site wired byte-parity-gated; the **Energized migration** (the
  `isEnergized` branch deleted from `commitInflict` — the stun baton now
  shocks through its own `augmentInflict`, pinned byte-identical); the
  `check-combat-dynamics` lint; **`CombatApi.influence`** +
  `Poise.exposeWindow`; **non-mechanical innates** (`commitShockInflict`,
  the electric eel armed); the **species combat vocabulary**
  (`Species.naturalAttacks[]`/`affordedGambits`, beat-keyed rotation, the
  body-derived natural profile replacing the five unarmed fallback
  literals); and the gym's hooked determinism cell. Four new dials
  (`combat.influence.*` ×3, `combat.natural.largeBodyMassKg`).
