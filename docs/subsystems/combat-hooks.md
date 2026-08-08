# Combat hooks — the extension grammar

The **wizard-facing extension tier for combat**: a declared, documented
roster of `@hook` seams — across three surfaces, at fixed points of the
beat lifecycle — through which custom TypeScript (special weapons,
reactive gear, species dynamics, venue responses) plugs into the combat
engine without reading its guts, and without the engine accumulating
per-dynamic carve-outs. The model is the movement substrate:
`Mobile.traverse` runs a fixed choreography of veto hooks, witness hooks,
and self-resolving scan points, so every movement dynamic is a named
method on the dynamic's own class and the engine never grows a branch.
This build gives combat the same property — **witness + compute hooks
only** (combat's vetoes already exist and stay where they are: the
`CombatTerms` handshake, attempt-time `CombatApi.eligibilityFor`, the
flee gate) — migrates the one pre-existing barnacle (the hard-coded
`isEnergized` shock branch in `commitInflict`) onto the grammar
byte-identically, and lint-locks the engine against the next one.

Built by the combat-hooks build; seeded by the 2026-07
content-extensibility audit (no slate — the requirements doc was the
first artifact). Wraps [combat](./combat.md) (the engine), models on
[spatial](./spatial.md) (the traverse choreography) and
[hazard](./hazard.md) (the marker-mixin scan), consumes
[electricity](./electricity.md) (the migrated first consumer) and
[call-security](./call-security.md) (shadows; why consequences are
context-mediated).

## The choreography

Every seam is an `@hook`-tagged method (canonical declaration authored
once; override sites match by name in the author-surface projection). In
beat order:

| Seam | Surface | Kind | Exact engine moment |
|---|---|---|---|
| `onWielded` / `onUnwielded` | instrument | witness | `Slotted.occupy` (after the successful slot add) / `vacate`·`vacateSole` (after the generic `onSlotReleased`) — **per slot** (a 2H claim hears it once per slot), outside any session (plain `(host, slotName)` args, no ctx). Persistence restore and clone hydration re-arm through the same chokepoint (`SlotApi.occupyAll`), so `onWielded` fires there too — bodies must be cheap and idempotent. |
| `onSessionEntered` | participant | witness | `openSessionImpl` success tail (initiator then defender) and `joinImpl` (the joiner), before the venue hook. |
| `onCombatOpened` | venue | witness | `openSessionImpl` success tail, after the participants; anchor = the initiator's room; a mid-fight `join` never re-fires it. |
| *beat top* | — | — | `advanceImpl`: band-baseline snapshot → interception pass → weapon-switch advance → brain intents → the reach-sorted exchange loop. Per-beat participant hooks fire in `getStates()` roster order (insertion-ordered), never the reach sort. |
| `augmentInflict(spec, ctx)` | instrument (the **carrier**) | compute | `commitInflict`, on the strike's augment carrier, before `ConditionApi.inflict`. Returns the (possibly reshaped) spec — any non-`shock` `InsultKind` (a flaming blade re-channels to `heat`); a malformed return falls back to the pre-augment spec with a warn, never a throw mid-beat. |
| *drain* | — | — | The engine applies the ctx's queued consequences from its own frame, in queue order, right after the primary inflict — the exact sequence position the deleted `isEnergized` branch occupied. |
| `onStruck(ctx)` | instrument (covering gear) | witness | `commitInflict`, on each CombatReactive item in the covering stack at the struck site (worn Constructed+Wearable armor on covering slots, the wielded shield when facing), outside-in, after primary + drain — whether the blow landed or was fully attenuated (`ctx.deflected` says which; "the shield took the blow" is the point). |
| `onParry(ctx)` / `onBypassed(ctx)` | instrument (defender) | witness | `resolveExchange`, immediately after `decideOutcome`: `parried` → the guarding instrument; `land` past a **steady but guardless** defender (the flail) → the defender's resolved instrument. `control-resisted` is not a parry — excluded; `exploit` is too (it needs the target's window open, never `steady` — bypass-at-steady is a land-only phenomenon). |
| `onExchangeResolved(ctx)` → `onStrikeResolved(ctx)` | participants, then the actor's carrier | witness | `witnessExchange`, once per exchange at each outcome-case tail — after any riposte (`reactiveDispatch`), so the witness sees the fully-resolved exchange; the riposte itself does not re-fire it. Participants actor-first, then the carrier hears its own land/whiff/parried (`ctx.outcome` set — combo/momentum dynamics). |
| `onPoiseBandChanged(ctx)` | participant | witness | `advanceImpl`, a **per-beat net transition**: compared after the tick loop against the cross-beat `bandSeen` baseline, fired in roster order (see § External influence for why the baseline is carried across beats). |
| `onBloodDrawn(ctx)` | venue | witness | The three `markBloodDrawn() === true` sites, via the shared `dispatchBloodDrawn`, after the narration roar. Exactly once per session. |
| `onDowned(ctx)` | participant | witness | Both `down = true` sites — `handleDown` (the poise-contest loss) and `checkVitalsResolution` (the attrition unconsciousness), after the stamp. |
| `onDefeated(ctx)` → `onDefeatedFoe(ctx)` → `onCombatResolved(ctx)` | victim, the named killer, venue | witness | `endWith`, after `narrateResolution`, **before** `session.resolve` — states still live. `onDefeatedFoe` fires only when a killer is named (the attrition path names it via `lastStruckBy`); anchor for the venue = the victim's room, else the first combatant's. |
| `onCoupBegun(ctx)` | participant (executioner, then victim) | witness | `startCoup`, only after `SchedulerApi.start(coup).ok`, alongside the telegraph. The session's states have already dissolved — `ctx.actorState`/`targetState` are null there (a late-ctx tolerance every post-resolution hook body must have). |

## The three surfaces

**Instrument — `CombatReactiveMixin`** (`lib/combat/CombatReactive.ts`,
`Mixins.CombatReactive`). The one marker mixin the engine scans for —
`MixinApi.isCombatReactive(item)` at the declared scan points, never
duck-typed method-presence checks, never per-dynamic `MixinApi.isX`
branches (the barnacle this grammar exists to kill). Seven hooks, every
default body a no-op terminal (the `onDestruct` shape) so overriders
compose via `super`; `augmentInflict`'s default is the identity. The
**carrier rule**: the augment carrier for a strike is the wielded weapon
when armed, else (innate/bare path) the attacker itself when it is
CombatReactive — **exactly one, never both** (the double-fire guard; a
venomous bite and a poison blade are the same abstraction with a
different carrier). The riposte runs the same rule for the defender's
instrument.

**Participant — `CombatantMixin` terminals** (`lib/combat/Combatant.ts`).
Seven no-op terminals (`onSessionEntered` / `onExchangeResolved` /
`onPoiseBandChanged` / `onDowned` / `onDefeated` / `onDefeatedFoe` /
`onCoupBegun`) on the mixin every combatant already composes — no new
marker, and **NPC≈PC parity is structural** (the same methods fire for
brain-driven and player combatants; a gym test pins identical hook
logs).

**Venue — the optional `CombatVenue` hooks** (declared on the
`CombatVenue` interface in `CombatHookContext.ts`). Three optional
members (`onCombatOpened?` / `onBloodDrawn?` / `onCombatResolved?`),
dispatched the way `traverse` dispatches `onEntered`/`onExited`:
presence-checked (`callVenueHook`), present → called, absent → skipped.
The base `Location` is untouched; a hook-less room (a gym
`ContainerMixin(Idea)` box, a non-Location container) is simply skipped.
A reactive venue is a Location subclass (or future mixin) overriding
them — the gate-barring arena, the crowd-reacting colosseum.

## `CombatHookContext` — read freely, consequence through the queue

Every ctx-taking hook receives one **`CombatHookContext`**
(`lib/combat/CombatHookContext.ts`, a named value-object — plain class,
never a Stuff, never proxied), so signatures never churn as the engine
evolves. Two faces:

- **The read surface** (readonly, nullable where a seam lacks it):
  `session`, `beat`, `actor`/`target` + their `CombatantState` views,
  `gambit`, `outcome` (the `ExchangeOutcomeKind` union — its canonical
  home moved here; `CombatLogic` re-aliases it), `channel`, `site`,
  `deflected`, `instrument`, `venue`, `resolution`. Hooks **read
  freely** — any participant surface (material, species, poise band) is
  directly readable; the silver-edge-vs-lycanthrope dynamic is a plain
  read plus a queued rider. Only *consequences* are mediated.
- **The consequence queue** — eight methods: `attachRider(spec, on?)`
  (the spec validated fail-fast at attach — a malformed hook-authored
  rider throws `TypeError` from the hook's own frame, never from the
  engine's drain), `afflict(condition, on?)`,
  `introduceToxin(type, amount, on?)`,
  `adjustReserve(key, delta, on?)`, `wearInstrument(amount, on?)`,
  `influence(instruction, on?)` (the § External influence vocabulary —
  hooks and the external bridge speak one economy), `deliverShock(source)`
  (target-fixed — a no-target ctx throws at queue time),
  `attachFlavor(line)`. The `on?: 'attacker' | 'defender'` recipient
  (default `'defender'`) resolves against the ctx's own exchange pair —
  recipient-constrained, never arbitrary targeting; queueing against a
  null resolved recipient throws.

**Why the funnel.** A hook never calls a gated Api from its own frame —
hooks must not become a permission bypass (arbitrary mixin code
acquiring `ConditionApi.inflict`-caller standing; the git-workflow
"never a permission bypass" doctrine). The engine constructs the ctx,
dispatches the hook(s), then **drains** from its own frame in queue
order — so provenance and accountability bookkeeping are byte-identical
to the engine's own calls (the inflicter still derives from the command
stack via `getActingAuthor`; blame rows, death causes, and trauma
attribution need no new writers). **Self-state mutation is sanctioned**:
a hook freely mutates its own host (combo counters, charge gauges, its
own wear) directly, under the determinism contract.

Drain arms: rider → `ConditionApi.inflict(recipient, spec)` (stamping
`lastStruckBy` when the recipient is the exchange target); afflict → the
recipient's own `afflict` door (`isVitals`); toxin →
`Metabolic.introduceToxin` (the hazard-dart seam, funneled); reserve →
the `Reserved` surface; wear → the recipient's wielded instrument's
`DurableMixin` gauge (no wielded instrument → inert); influence →
the same module-private `influenceImpl` as `CombatApi.influence`; shock
→ `ElectricityApi.shockContact(source, target)` (the `effectiveVoltage
≤ 0` guard inside the conduction walk still applies — a switched-off
source truthfully delivers nothing); flavor → buffered and emitted
through `CombatNarration.narrateFlavor` (the existing witness loop)
**after** the exchange's own narration beat, in queue order.

**The sealed-context rule**: `_drain()` runs exactly once, from the
engine, and seals the ctx — every subsequent `attach*` throws (closing
the stashed-context determinism hole). A rogue hook-constructed context
is inert: only the engine drains the one it built. **Uniform drain
rule**: every dispatched ctx is drained — witness hooks carry
consequences too (a fear aura afflicts at `onSessionEntered`).

## The determinism contract

Binds **every** hook body — override or shadow: **synchronous,
deterministic, cheap** — no `await`, no wall-clock, no randomness,
bounded work per beat (hooks run inside the real-time session tick).
Witness-hook throws are guarded (warn-and-continue — an author's broken
witness never kills the beat for everyone; the compute hook falls back
to the pre-augment spec). Routing hook-author failures through
`DiagnosticApi.record` so they reach the author is a named deferral.
Stated in every `@hook` TSDoc; enforced by the gym, not a runtime guard:

- **Byte-parity default** (the standing gate, not a one-time check):
  with no CombatReactive gear and no overrides present, every session is
  bit-for-bit identical to pre-hook combat — the full gym PINS table
  (feinter/brain/spear/swordshield cells, the stun-baton migration pin,
  the unarmed-innate pin) runs in every suite.
- **Hooked sessions stay deterministic**: the gym's hooked determinism
  cell (a test-scoped `CombatReactiveMixin(Weapon)` blade attaching a
  fixed-energy `tearing` rider, witnesses counting fires) runs twice to
  identical winner, beats, **and hook-fire counts** — alongside the
  untouched PINS in the same run. The rider is damage-side, not
  contest-side: like the stun-baton's shock, it leaves the pinned
  mechanical trace byte-identical while the loser's condition roster
  carries its mark (the roster is where a rider-shaped dynamic is
  visible; a poise-touching dynamic — an `influence` consequence — moves
  the trace itself, deterministically).

## Author patterns (the coverage-exercise cookbook)

Validated across ~40 candidate dynamics walked through the seams
(requirements rounds 1–3):

- **Read freely, consequence through ctx, self-state sanctioned** — the
  three-clause base pattern above.
- **Deterministic procs** — never RNG (poker, not slots; the
  chance-on-hit analog): an every-Nth-hit combo counter is TS-private
  self-state incremented in `onStrikeResolved` and cashed in
  `augmentInflict` when the threshold hits; conditional dynamics are
  threshold reads over the ctx (poise band, outcome, deflected), never a
  roll.
- **Session-keyed transient state** — a session-hardening shield keys
  its per-fight state on `ctx.session` (a plain map on the host,
  cleared/superseded per session); nothing persists.
- **The oil pattern** — a consumable applies a **timed shadow** over the
  weapon's hook (attach at apply, detach on expiry via the scheduler) —
  the Witcher-oil shape, zero new machinery (§ Enchantment below).
- **Shadow stacking order = attach order** — the Hades-boon shape: the
  shadow chain over one hook runs top-down from the most recent attach;
  composition is the chain, each layer choosing `callDown`.

## The first consumer — the Energized migration

The proof-by-construction worked example. **Before**: `commitInflict`
held the barnacle — `if (weapon && MixinApi.isEnergized(weapon))
ElectricityApi.shockContact(weapon, target)` — combat knowing about
electricity. **After**: `EnergizedMixin` composes `CombatReactiveMixin`
(a nested factory — `hasMixin`'s per-level `_mixinName` walk narrows the
composition as both `Energized` and `CombatReactive`) and overrides
`augmentInflict`:

```ts
augmentInflict(spec, ctx) {
  ctx.deliverShock(this);
  return super.augmentInflict(spec, ctx);
}
```

The branch is **deleted** (`MixinApi.isEnergized` no longer appears in
`CombatLogic` — the lint keeps it out); the drain sits at the same
sequence position the branch occupied, the `effectiveVoltage ≤ 0` guard
is unmoved (a switched-off baton truthfully delivers nothing), and the
gym's `stunbaton-vs-sword` pin — winner/beats **plus** the loser's
shock-burn condition roster — was captured against the pre-migration
engine and holds byte-identically across the flip. `StunBaton`'s
composition is unchanged (`CombatReactive` arrives via `Energized`).

## Enchantment via shadows — the verified boundary

Shadows intercept named methods dispatched through the security proxy;
every hook is exactly that — so a temporary enchantment is a shadow over
`augmentInflict`, a curse is a shadow over a participant hook, and
detach ends the effect with **zero new machinery**. The verified
mechanics (the `CombatReactive.shadow` suite): the shadow chain
**fully replaces** the base call — an enchantment that wants the base
behavior (almost always: `augmentInflict` must preserve the host's own
`super`-composition) continues via `Shadow.callDown`; attach is
proxy-keyed, so the engine's own dispatch runs the enchantment;
detach restores baseline **byte-identically**; a shadow body is bound by
the same determinism contract and the same context-mediated consequence
rule (the drain runs in the engine frame regardless of the shadow
frame's target). Consequently **no hook is ever marked `@Final` or
`@Unshadowable`** — the shadow chain over a hook *is* the
temporary-enchantment substrate.

**The empirical dispatch boundary** (found in Phase 2, pinned by test):
a shadow **reshapes existing hooks but cannot ADD one**. Instance-level
`hasMixin` walks attached shadows (the Witness pattern), so a
`CombatReactiveMixin`-composing shadow lights
`MixinApi.isCombatReactive` on a mundane `Weapon` — the narrowing turns
true — but the proxy's get-trap returns `undefined` for a method the
host itself doesn't define, so the engine's presence guard (`hookFn`)
skips the absent hook **silently**: the shadow body never runs, the
fight stays byte-baseline, nothing throws. The **working enchantment
shape** is therefore a shadow over a host that **composes the mixin**
(the no-op terminals are exactly the cheap base a shadow reshapes).
Enchant-*anything* would require composing `CombatReactiveMixin` into
the base `Weapon` (no-op terminals cost nothing) — a named future sweep,
deliberately not built this cycle.

## External influence — `CombatApi.influence`

The gated bridge for systems *outside* the engine (a future spell, the
script interpreter, a director) to issue **state instructions** into a
live fight — "Effect iff gated Api" made real for combat state (a
firebolt could always wound; nothing could stagger until this surface
existed). `CombatApi.influence(combatant, instruction): {ok, reason?}`
takes the closed `CombatInfluence` union (`lib/combat/
CombatInfluence.ts` — `stagger` light|heavy / `expose` / `steady`;
`ok:false` reasons `not-in-combat` / `downed` / `suppressed`). The
facade/logic gate is the standard combat-mutator shape — callers reach
it from their own gated logic; **no spell ships here** (the magic track
consumes it).

Every instruction routes through the **existing poise/opening economy**,
never a parallel one:

- **`stagger`** → `poise.erode` at the banded dial
  (`combat.influence.staggerLightErode` / `staggerHeavyErode`), scaled
  by the same focus-fire edge-count multiplier the exchange applies — an
  already-pressed target staggers harder. A crossing into `broken` arms
  the normal opening; influence never sets `down` (only an exchange
  exploiting the contest does).
- **`expose`** → `Poise.exposeWindow` (the one additive `Poise`
  mutator: arms the binary timed window without moving the gauge,
  idempotent while open) — the window then ticks, lapses, and is
  consumed by the existing exploit machinery. `openingArmedBy` stays
  **null**: an external window is ownerless — any attacker cashes it,
  and no `command`-Discipline deed is minted for it.
- **`steady`** → `poise.restore` at `combat.influence.steadyRestore`,
  endurance-capped like any recovery and **suppressed under the
  focus-fire pin** exactly like the defend verb — influence can't
  out-recover a gang-up the verb can't.

**The `bandSeen` baseline.** Band changes influence causes are
witnessed by the existing `onPoiseBandChanged` — no new hook — via a
build reality note: the per-beat comparison baseline is not a pure
beat-top snapshot but the **cross-beat `bandSeen`** stamp (the band as
of the previous beat's dispatch, falling back to a top-of-beat read on a
state's first beat). Byte-identical to the snapshot shape for anything
that mutates poise mid-beat — carrying it across beats is precisely what
lets a **between-beat** external mutation surface on the following beat
instead of vanishing into the next snapshot.

## The species vocabulary

The authorable `Species` face of the same grammar (see
[race.md](./race.md) for the Species Idea itself):

- **`naturalAttacks[]`** — `Species.naturalAttacks: NaturalAttackSpec[]`
  (`@authorable`; `lib/combat/NaturalAttack.ts` — `{key, channel,
  reach?, massKg?, lengthM?}`). Multi-attack variety (claw/claw/bite)
  manifests as **deterministic beat-keyed rotation**: the striking
  attack is `attacks[(session.getBeat() − 1) % n]`, read from the
  session beat and nowhere else (a per-state counter would drift under
  tempo variance and break two-run transcript equality);
  **presence-only reads** (attempt-time eligibility, guard) use entry 0
  — they run at un-beat-anchored moments. The legacy
  `CombatantMixin.naturalAttackChannel` stays as the single-attack
  fallback, synthesized as a one-entry list (key `'natural'`, no hints)
  — byte-preserving, gym-pinned.
- **The derived natural profile** — `NaturalAttack.deriveProfile(spec,
  bodyPlan, config)` yields the four numbers the unarmed fallbacks used
  to hard-code (`tempoFactor`/`poiseDamageFactor`/`overextendFactor`/
  `reachRank`), consumed via the engine's `actorStrikeProfile` (weapon
  profile when armed, else the current natural attack's profile, else
  neutral). Authored hints ride the **same `WeaponProfile` curves** as a
  wielded weapon (zero new curve dials); hint-less derivation is a
  banded body-scale term from `BodyPlan.baseMass` — **exactly neutral
  `(1, 1, 1, 0)` below `combat.natural.largeBodyMassKg`** (seeded 150 —
  wolves ~40 kg and humanoids ~70–90 kg stay neutral; the unarmed gym
  pin is the tripwire), and at/above it one reach rank + heavy-balance
  factors via an **effective limb** of mass `balanceRefMass × (bodyMass
  / largeBodyMassKg)` — **continuous at the threshold** (exactly the
  neutral reference mass at 150 kg, so the balance factors don't step;
  only the reach rank does). An ogre punches at ogre reach; a future
  body seeded past 150 kg changes combat feel *by design*.
- **The non-mechanical delivery split** — `resolveInstrument`'s innate
  branch returns any `Channel`; `commitInflict` splits on it exactly as
  the weapon doctrine states: mechanical → today's `EnergyInflictSpec`
  path byte-identically; `heat` (and any non-shock `InsultKind`) → an
  energy spec through the insulation fold/passthrough; **`shock`** → no
  mechanical primary at all (`commitShockInflict`): the innate carrier
  rule fires `augmentInflict` on the creature, the `Energized` override
  queues `deliverShock(this)`, and the drain delivers the one shock —
  **single-fire by construction** (the split never calls `shockContact`
  itself). Two build reality notes: the spec handed to the shock-path
  hook is **descriptive only** (a placeholder `current: 0 A` — the real
  current is unknowable until the drain's conduction walk resolves the
  contact — and the hook's return is discarded; there is no primary to
  reshape), and the **parting shot is a documented no-op for a shock
  innate** (`partingShot` returns on `channel === 'shock'` — its
  delivery is the contact drain, not an energy spec, and the fleer is
  breaking contact; a parting shock is a deferred seam). The **electric
  eel** is the seam's second consumer: a creature class composing
  `EnergizedMixin` directly (never via `Species.innateMixins` — that
  conferral is activation-gated, the documented trap) with a `shock`
  natural attack; an `Energized`-less shock innate truthfully deflects
  (the surfaced authoring error). The `InflictReport` for a shock-only
  strike derives from the drained outcome (landed → the band over
  `MaterialApi.resolveShock`'s severity; `lastStruckBy` stamped).
- **Species-afforded gambits** — `Species.affordedGambits: string[]`
  (`@authorable`, **existing gambit kinds only** — the closed-vocabulary
  non-goal stands; a bogus key is inert). `speciesAffords` short-circuits
  the **two equipment gates only** (`affordedByForm`/`affordedByShield`)
  in `eligibilityImpl` — a tailed species affords `sweep` bodily;
  `needsInstrument` still stands (satisfied by the natural attack), and
  every other gate (down, terms, injury) is untouched.

## The lint — `check-combat-dynamics`

`scripts/check-combat-dynamics.ts` (the `check-inert-weapon`/
`check-does-nothing` script shape; `pnpm -C packages/server
lint:combat-dynamics`, CI-wired): scans `obj/api/CombatLogic.ts` +
`lib/combat/*.ts` (tests excluded) and flags any `MixinApi.is*`
narrowing outside the **21-predicate physics allowlist** — the substrate
predicates (slots, containment, vitals, construction, party, senses…)
plus the drain's consequence narrowing (`isMetabolic`/`isReserved`/
`isDurable`) and `isWearable` (`coveringGearAt` mirrors
`resolveCoveringStack`'s item selection). `isEnergized` flags from
either home — the migration stays locked. **When it fires: implement a
hook, don't grow the allowlist** — a *dynamic* predicate (isEnergized,
isVenomous, …) is never physics; adding a predicate is a deliberate
review-visible edit with a one-line reason.

## The author surface

Every seam carries `@hook` and lands in the **extension** tier of the
generated author surface (`pnpm docs && pnpm docs:project` —
`callable == visible == cared-about` extended to wizards). The
build-time projection finding was the **inverse** of the anticipated
risk: the three `CombatVenue` **interface** members projected cleanly,
while the mixin **class** methods did not — a mixin factory's returned
class expression is not walked by the projector (the module export is a
function, not a class/interface reflection; the same TypeDoc limit
behind the `@authorable` field-schema scan). Resolution (the plan's
fallback, mirrored): the projected `@hook` contracts live on the
**`CombatReactive` and `Combatant` interface members** — the interfaces
are the canonical projected declarations, the mixin method TSDoc is the
in-editor twin carrying the full composition mechanics. All seventeen
seams (7 instrument + 7 participant + 3 venue) land in the extension
tier.

## Dials

| Key | Default | Meaning |
|---|---|---|
| `combat.influence.staggerLightErode` | 0.12 | Light stagger erosion (the exchange-erosion literal) |
| `combat.influence.staggerHeavyErode` | 0.3 | Heavy stagger erosion |
| `combat.influence.steadyRestore` | 0.15 | The `steady` recovery assist (the defense-restore literal) |
| `combat.natural.largeBodyMassKg` | 150 | Body mass at/above which a hint-less natural attack derives large-body reach + heavy balance |

The hook **grammar** itself carries zero settings — dynamics bring their
own dials (magnitude-in-dials per doctrine); these four belong to the
influence bridge and the natural-profile derivation.

## Graduation posture (recorded, not scoped)

When two or three coded dynamics converge on a shape (several
`augmentInflict` overrides that just attach an authored Condition), that
shape graduates to a data-driven mixin with an `@authorable` field — the
magic-`Effect`-union path. The hook grammar is what makes that fold
cheap; this build deliberately pre-built none of them. The
consumable/item-effect envelope is the sibling magic-items build.
Named deferrals (clean attach points, not stubs): defense-side compute
(`augmentMitigation` — materials-response territory), the worn-gear
multi-carrier augment chain (the totem), ally-aura recovery modifiers,
a generic `onSlotOccupied` twin, the material-affinity layer,
battle-magic itself, composing `CombatReactiveMixin` into base `Weapon`
(enchant-anything), the parting shock.

## Proof

Unit suites: `CombatReactive.test` / `CombatHookContext.test` (substrate
contracts, seal, recipient constraint), `CombatLogic.hooks.test` (every
seam observed through real fights: reshape-through-inflict, riders both
directions + `lastStruckBy` discipline, toxin/reserve/wear/influence/
shock/flavor drains, covering-stack `onStruck` incl. deflection, the
double-fire guard, participant + venue witnesses, NPC≈PC hook-log
parity), `CombatReactive.shadow.test` (the enchantment round-trip + the
dispatch boundary + shadowed determinism), slot-chain tests (per-slot
`onWielded`/`onUnwielded` incl. the restore-path fire), the Energized
migration ordering test, `CombatInfluence`/`Poise.exposeWindow` suites,
`NaturalAttack.test` (neutral-band + threshold continuity + rotation).
The **gym** carries the standing gates: the full PINS table (stun-baton
+ unarmed pins included), the hooked determinism cell, the influence
fixed-beat cell, the ogre-reach cell — byte-parity defaults and live
hooked/influenced/derived variations deterministic side by side.

## Cross-references

- **Wraps:** [combat](./combat.md) — the engine and its beat; the
  pointer section there leads here.
- **Modeled on:** [spatial](./spatial.md) (`Mobile.traverse`),
  [hazard](./hazard.md) + [residency](./residency.md) ("engine informs,
  object decides"; no-op terminals).
- **Consumes / consumed by:** [electricity](./electricity.md) (the
  Energized migration; `shockContact`), [harm](./harm.md) +
  [materials-response](./materials-response.md) (the inflict funnel the
  riders ride), [race](./race.md) (the `Species` authoring surface),
  [call-security](./call-security.md) (shadows, gates, why consequences
  are context-mediated), [app-settings](./app-settings.md).
- Future consumers: the magic track (`CombatApi.influence`), the
  magic-items envelope (data-driven dynamics).

## History

Built as the combat-hooks build (branch `feature/combat-hooks`,
`f0120414..`, phases 1–9): the substrate (mixin + context + venue
interface), the engine wiring at every dispatch site (byte-parity
gated), the Energized migration (pin-then-flip), the
`check-combat-dynamics` lint, `CombatApi.influence` + `Poise.
exposeWindow`, non-mechanical innates (`commitShockInflict`, the eel),
the species vocabulary (`naturalAttacks[]` rotation + the derived
natural profile + species-afforded gambits), and the gym's hooked
determinism cell. Two empirical findings reshaped the design mid-build:
the **shadow dispatch boundary** (shadows reshape, never add — § the
enchantment boundary above), and the **`bandSeen` cross-beat baseline**
(a pure beat-top snapshot would swallow between-beat influence — § the
influence bridge). The optional demo-dynamics items (bramble mail /
ember edge) were deliberately not built — clean follow-ons on a proven
grammar.
