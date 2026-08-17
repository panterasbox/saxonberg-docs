# Harm

The **injury driver** over the [vitals](./vitals.md) substrate: the piece
that lets an external insult wound a body, makes that wound progress or
heal over game-time, and lets a wound kill. Vitals shipped the *models*
(the `Trauma` value, the anatomy, the death/consciousness seams, the
`TRAUMA_BEHAVIOR` table skeleton with every entry no-op); harm fills in the
parked *behaviors*. It is the fourth instance of the established driver
pattern ([metabolism](./metabolism.md) / [thermal](./thermal.md) /
[respiration](./respiration.md)) and the keystone on the combat critical
path, but it **stands alone without any combat loop, weapon, or armor**: a
body becomes woundable by ordinary hazards, and a non-combat **medic** loop
(assess → treat, skill-gated) is playable before combat exists.

## `ConditionApi` — the condition-surface facade

The gated facade over the vitals **condition** surface (`api/condition.ts` →
the gated `ConditionLogic` singleton at `/obj/api/condition`). Reserves
(endurance) and transient combat flags are NOT conditions and stay out.

> **Corrected 2026-07-31 (verified against the code).** This section used to
> claim the facade also "forwards the plain condition mutators —
> `afflict(target, condition)` / `relieve(target, condition)` — and a query
> `conditionsOf(target)`." **It does not.** `api/condition.ts` has **exactly
> one** public static, `inflict`, and `ConditionLogic` has exactly one
> `@CallSecurity`'d method, also `inflict` — which is **trauma-only**
> (`InflictSpec` is a discriminated union over `mechanism` and always mints a
> `Trauma`). *(The stale wording also survives in `condition.ts`'s own header
> comment.)*
>
> **Consequence for anyone building an affliction driver** (disease, poison, a
> new cascade): **an affliction cannot be inflicted through the Api at all.**
> Follow the shipped driver precedent — metabolism, respiration, thermal, and
> magic all call the body's own `VitalsMixin.afflict()` / `relieve()` directly
> from their mixin — or add a gated `ConditionApi.afflict` first. See
> disease-slate.

The Api is deliberately **bounded**: internal drivers keep calling the body
methods directly; it ADDS a producer facade, it does not re-route them.

## The `inflict` producer

`ConditionApi.inflict(target, spec) → InflictOutcome` is the **single seam
every harm source calls** — this build's floor-glass hazard, and later
combat. It builds a `Trauma`, lands it through the existing
`VitalsMixin.afflict()` door, runs the trauma's `onset`, and stamps the
reconcile-on-read `tickedAt` anchor (see below — no arming).

`InflictSpec` is a **discriminated union** (added by the
[electricity](./electricity.md) build): the energy variant
`{ mechanism, site, energy }` (mechanical channel or thermal/tearing
passthrough) vs the shock variant `{ mechanism:'shock', site, current }` —
whose magnitude is the *current through the victim* (`Quantity<'A'>`), not an
energy.

- **Gated producer.** `inflict` is a powerful primitive that must not be
  callable by arbitrary content. `ConditionLogic.inflict` carries
  `@CallSecurity(FromModule('/api/condition#ConditionApi'))` — only the
  `ConditionApi` facade forwards in; trusted producers (the hazard, later
  combat) reach it through the Api. The inflicter is un-spoofable, so
  `ConditionApi.inflict` itself stays reachable (the
  `BulletinApi`/`ProvenanceApi` precedent).
- **Inflicter from context.** The inflicter's durable `templatePath` is
  derived from `ExecutionContextApi.getActingAuthor()` (command-frame giver
  when non-forced + single-consistent, else the REST acting-author stamp,
  else `undefined`) — **never a caller-supplied parameter** (the gated-Api
  actor-from-context rule). Recorded on `Trauma.inflictedBy` for combat's
  future blame ledger; harm records attribution without owning blame.
- **Severity AND type from the response function.** As of the
  materials-response build, `mechanism` is an `InsultKind` — a
  materials-response **`Channel`** (`edge`/`point`/`blunt`) or a legacy
  passthrough token. For a `Channel`, `inflict` resolves the covering stack
  at `site` outside-in, attenuates the incident `energy` through each armor
  layer, and lets the residual meet the site's tissue — yielding **both**
  the `Trauma` *severity* AND its *type* (edge→laceration, point→puncture,
  blunt→fracture-on-bone-else-contusion; a fully-attenuated blow lands no
  wound, a truthful "deflected"). The old magnitude-only `severityFromEnergy`
  + bijective `mechanismToType` switch is retired for channels; it survives
  only on the **`'thermal'`/`'tearing'` passthrough** (direct → burn /
  avulsion), the documented seam that folds into a future `heat` / tearing
  channel. `mechanism` is still **recorded raw on `Trauma.mechanism`**. The
  response function is the single `MaterialApi` chokepoint — see
  [materials-response.md](./materials-response.md).
- **The `shock` third path.** A `mechanism:'shock'` insult is intercepted
  **before** the covering-stack fold and routed to a third path that maps the
  current-through-victim straight to a contact `burn` (`MaterialApi.
  resolveShock`) — the path resistance was resolved upstream by the
  conduction walk, so shock **skips the fold** (metal armor does not protect).
  The mechanical fold + the thermal/tearing passthrough stay byte-identical.
  See [electricity.md](./electricity.md).

## The five trauma behaviors

Live in `lib/vitals/Condition.ts`, co-located with the `Trauma` value in
the closed `TRAUMA_BEHAVIOR: Record<TraumaType, TraumaBehavior>` roster.
The interface is `onset` / `tick(host, t, elapsedSec)` / `resolve` (the
*dress* action) / `reopen` (the *undress* action) / `describe`. Rates live
in the `HARM_DEFAULTS` const-object (the driver `*_DEFAULTS` convention).

- **laceration** (flagship) — `onset` opens the bleed (`bleeding = true`);
  `tick` while bleeding-and-undressed drains `bloodVolume`
  (`BLEED_PER_SEC · severity · elapsedSec`; an open bleed does **not**
  self-clot — you must dress it), and once dressed OR clotted-open decays
  severity toward clear (fast while `dressed`, slow otherwise). The **clot
  gate**: `resolve` (dress) sets `dressed`, arrests the bleed; `reopen`
  (undress) clears `dressed` and re-arms `bleeding` iff `severity >
  HARM_DEFAULTS.CLOT_SEVERITY` (below it the wound has clotted, safe to
  remove). The gate is a pure boolean-flag machine on the `Trauma` value —
  no dependency on the `DressingMixin`, the `Bandage`, or the verbs (those
  are the consumers that call `resolve`/`reopen`).
- **contusion** — mild, self-resolving severity decay; no bleed.
- **fracture** — a slow natural heal. **The impairment is a derived read**,
  not a tick effect: `Vitals.isSlotImpairedByTrauma(slot)` folds into
  `Slotted.canOccupy` alongside the anatomy gate, so a fracture at a slot's
  `bodyPart` above `FRACTURE_IMPAIR_SEVERITY` greys that slot's
  affordances — and healing/clearing the fracture restores them with no
  separate un-impair step.
- **burn** — real behavior: severity + a slow heal at its own rate.
- **avulsion** — behaves as a **severe laceration** (floors severity via
  `AVULSION_SEVERITY_FLOOR`, bleeds, shares the clot gate). ► **Deferred
  seam — the limb-sever / part-promotion** (mark the `BodyPart` missing,
  cascade slot-disable + presentation) lands at `AVULSION_BEHAVIOR.onset`
  when the sever build arrives; v1 stops at the severe bleed.

## The wound driver — reconcile-on-read

Wound progression is **reconcile-on-read**, exactly like its sibling
drivers (`Metabolic.reconcileMetabolism` / `ThermalRegulation` /
`Respiration`) — **not** a recurring push tick. There is **no
`ScheduleApi.recurring`, no in-memory tick-handle map, and no re-arm
seam**. The driver lives on the body itself:
`VitalsMixin.reconcileConditions()`, a private method run at the top of the
reads that must reflect the current bleed — `getVitalSign('bloodVolume')`,
`getConditionBand`, `getConsciousness`, `getConditions`.

- **The stamp persists, not a handle.** Each active `Trauma` carries a
  persisted game-time `tickedAt` anchor (rides the `VitalsMixin.conditions`
  collection). `inflict` stamps it at onset; every read advances it. A body
  coming live simply resumes from its last stamp on the next read — nothing
  to re-arm, no `Avatar.enter` / `NPC.postRegister` touch. (The old
  `HarmApi.rearmWoundTicks` seam is **gone**; `NPC` reverts to the bare
  `BehavedMixin` `postRegister`.)
- **Per-trauma integration.** For each active trauma,
  `reconcileConditions` computes the in-session game-time elapsed since its
  `tickedAt`, calls `TRAUMA_BEHAVIOR[t.type].tick(host, t, elapsedSec)`,
  then relieves any wound healed to (near) zero severity and re-stamps.
- **Presence-freeze parity.** The integration copies the
  `reconcileMetabolism` discipline: first-touch stamp, linkdead re-stamp
  (`isHasInteractive && isLinkdead`), `elapsed <= 0` guard, and the
  far-past guard (`MAX_REASONABLE_GAP_SEC`, 4h — a logout/relog gap
  integrates nothing). Cheap no-op when no world clock runs (unit tests
  stay idle) or no trauma is active. A `_reconcilingConditions` reentrancy
  guard keeps the vital-sign reads the method performs from re-triggering
  it.
- **Death by exsanguination.** After integrating, if `bloodVolume` is
  at/below its `survivableMin`, harm stamps its own death —
  `setCauseOfDeath('exsanguination')` + `setLifecycleState('dead')`
  (idempotent-guarded, the metabolism/respiration shape; there is **no**
  shared `applyDeath` helper). The `conscious → unconscious` waypoint needs
  no code: `getConsciousness()` already reads a low `bloodVolume` as
  `unconscious`.

## The couplings — limp + coverage

- **The limp** (`Vitals.drainForLimp`) is a severity-gated `endurance`
  drain summed over active locomotor (`body.leg.*`, incl. `.foot`)
  laceration/avulsion wounds, composed in at
  `LocomotionLogic.engageAround` alongside the encumbrance drain — the
  universal self-powered traverse chokepoint. Conveyance riders and
  raw/forceMove traverses skip it structurally. Eases as the wound heals.
  Distinct from fracture's slot-disable (that's a read; this is a movement
  cost).
- **Coverage is degree, not presence.** The binary
  `ConditionApi.isSiteCovered` read is **retired** (materials-response): the
  mitigation *curve* now falls out of `inflict` resolving the covering stack
  outside-in through `getSlotsCovering(partKey)` (the `covers` edge) →
  `Constructed`+`Wearable` armor layers → the tissue. A covered site turns
  what an uncovered gap takes, by material + construction + wear, not by a
  presence flag. See [materials-response.md](./materials-response.md).

## The medic vertical

Harm is the **first non-combat advancement consumer**. A new `medical`
command category plus `assess` in `perception`.

- **Dressing is a capability, not a class.** `DressingMixin` (`lib/vitals/`,
  `Mixins.Dressing` + `MixinApi.isDressing`, the `ToolMixin` pattern)
  carries a 0..1 `dressingQuality` and is **single-use**. `Bandage =
  DressingMixin(Thing)` (`obj/Bandage.ts`, the `Coin`/`BrandedBottle`
  precedent) is the canonical concrete one; any dressing-capable item
  (gauze, a rag) qualifies. `treat`/`undress` gate on `isDressing`, **never
  `instanceof Bandage`**. The instrument (splint/suture → `ToolMixin`) and
  medicine (→ bulk/metabolism) first-aid branches are named-but-deferred
  seams — no sibling mixins in v1.
- **`treat` / `bind` / `dress`** (`cmd/medical/`, `mustHaveDressing`
  validator) dresses a body's worst bleeding wound, consuming a reachable
  dressing (`StuffApi.destruct`) and calling the trauma's `resolve` (sets
  `dressed`, arrests the bleed); the dressed wound heals to clear on the
  next read (reconcile-on-read — no tick to re-arm). Outcome quality = the
  dressing's `dressingQuality`
  × the treater's `medicine` competence band; difficulty is derived from
  the wound (a world-measurement, not a tag). A graded outcome mints an
  `ActSignature` (`AdvancementApi.recordDeed`) into the treater's
  Transcript — consuming the advancement API, reshaping nothing in it.
- **`undress`** (a distinct verb from the wearable-slot `remove`) is the
  clot gate's other half: calls `reopen` (a premature removal above
  `CLOT_SEVERITY` re-opens the bleed; after clot it is safe). A re-opened
  bleed drains again on the next read (reconcile-on-read — no tick to
  re-arm). The bandage is spent, not recovered.
- **`assess`** (`cmd/perception/`) is a perception-gated readout, not a
  tool-mediated measurement (no stethoscope — deferred). Full fidelity on
  one's own body; banded + competence-sharpened on others (novice reads
  the qualitative gist, proficient+ reads precise severity). A **dressed**
  wound reads "bleeding controlled" and **hides precise severity behind the
  dressing** — an expert can judge through it, else `undress` and look.
  This is the loop that answers "how do I know I'm healed."
  It also carries an **affliction readout** — what is wrong with someone
  that isn't a wound. Trauma was the only thing it could describe, so
  anything *carried* rather than *cut* (a poison, a disease, the mortality
  floor's `recovering`) showed up nowhere and a body under it read
  "unhurt". Signs before names, on the same competence rule: `unsteady`
  for a novice, the condition's name for a competent medic. See
  [mortality.md](./mortality.md) for the `Condition`-Idea fallback it
  degrades through.
- **medicine `Discipline`** — an authored Catalog leaf
  (`seeds/lib/advancement/Discipline/medicine.yaml`, `key: medicine`,
  ISCED-F `0913`).

## The demonstrator — GlassAlley

A **one-off demonstrator room class** (`domain/lounge/GlassAlley`),
deliberately **NOT a reusable `HazardMixin`** — a capability mixin buys
nothing here (no multi-host reuse, no consumer narrows on it, no
composition; the reusable abstraction is `inflict` itself). It overrides
`onEntered(mover, exit)` (the `Mobile.traverse` presence trigger — NOT a
teleport arrival): resolve a foot site from the mover's own anatomy (a
non-biped matches none → graceful no cut) and cut the foot through `inflict`
with an `edge` insult (never `afflict` directly). There is **no explicit
coverage gate** any more — the materials-response covering stack decides:
a bare foot lacerates, a stoutly-shod foot's boot layer attenuates the edge
below the no-wound threshold. Config (mechanism / energy / foot sites) is
class constants. It proves the full loop end-to-end — step on glass →
bleed + limp → assess → treat-or-die — through the real `Mobile.traverse`
+ the medic controllers + the reconcile driver, in the **`GlassAlley`
integration test** (`domain/lounge/__tests__/GlassAlley.integration.test.ts`,
which constructs the alley + a body + an `Exit` in-memory). It is **not
wired into the world seed graph** (see the note below); the
reachable-in-world demo is deferred until a safe walkable host exists. A
real hazard/trap taxonomy is a separate future build over the same seam.

> **Shipped (concealment build): `HazardMixin` generalizes this seam.**
> The real hazard/trap taxonomy landed — a self-resolving `HazardMixin`
> (`lib/hazard/`, no `HazardApi` — the powerful steps route through the
> already-gated `ConditionApi.inflict` / `PerceptionApi.perceives`), a
> `HazardDelivery` value-object producing the `InflictSpec` (armor mitigates
> for free through the covering stack, as it does a blow), and the trigger
> fired from `Mobile.traverse`. See [hazard.md](./hazard.md). **`GlassAlley`
> stays separate on purpose**: it is an *obvious* underfoot hazard — always
> perceived, therefore always avoided — which is a different case from a
> *spottable, concealed* trap (`HazardMixin` resolves against the detection
> gate). Migrating GlassAlley onto `HazardMixin` is an optional, skippable
> proof; it remains a class + integration fixture so unavoidable-glass and
> spottable-trap coexist as distinct demonstrators.

> **In-world placement deferred.** The demo was briefly wired off Dave's
> Bar, then the Terminus Terminal hall, but every real content-area host
> broke a standup/fast-travel invariant (a cross-domain exit fails the
> terminus standup's isolated boot; the lounge's landing host asserts its
> exit count; the TPA terminals pull the room into the fast-travel
> cascade). Rather than degrade a tested content area for a demo, the
> seed wiring was retired — `GlassAlley` stays a class + integration
> fixture. Re-home it to a purpose-built, un-asserted walkable room when
> one exists.

## Deferred (named seams)

- **Materials-response severity function** — ✅ **LANDED**: `inflict`
  resolves severity + type through the covering stack into the tissue; the
  binary coverage gate is retired. See
  [materials-response.md](./materials-response.md). (Still deferred *within*
  materials-response: the tissue-vulnerability term, combat playstyle, the
  repair economy, other channels.)
- **Avulsion sever / part-promotion** — at `AVULSION_BEHAVIOR.onset`.
- **Combat resolution** — attack verbs, poise, blame ledger,
  `combat.body.*` events, offensive `ActSignature`
  (combat-slate.md); harm
  provides the substrate combat writes into and the `inflict` door it
  calls.
- **Instrument + medicine first-aid branches**, fuller consumable-crafting
  supply chain, metabolism-coupled healing.


> ⭐ **A condition's mechanism carry a reveal level.** `progression`, `resolution`, `contagion`, `toxinBehavior`, `signature` and `mentalBands`. `observableSigns` stays open — a sign nobody can read is not a sign, and the diagnosis loop runs on it. `spoiler: 1` with
> `spoilerName: 0`, so a reader sees which properties exist with the
> values collapsed — **one click, not a lock**, and a reader who set
> `wiki.spoilerAppetite` higher never sees a collapse at all. The rule
> the sweep settled on: *collapse what the WORLD measures, never what
> the PLAYER operates.* Levels are declared on the field and therefore
> apply wherever the value surfaces. See
> [wiki.md](./wiki.md) § the reveal model.

---

## Cross-references

- [vitals.md](./vitals.md) — the substrate: `Trauma`, anatomy, the
  death/consciousness seams harm drives
- [metabolism.md](./metabolism.md) / [respiration.md](./respiration.md) /
  [thermal.md](./thermal.md) — the driver precedent (reconcile,
  presence-freeze, death seam, `*_DEFAULTS`)
- [advancement.md](./advancement.md) — the `Discipline` / Transcript /
  `ActSignature` the medic vertical consumes
- [locomotion.md](./locomotion.md) / [encumbrance.md](./encumbrance.md) —
  the `LocomotionApi` seam the limp rides
- [slot.md](./slot.md) / [embodiment.md](./embodiment.md) — the worn-
  coverage presence check
- combat-slate.md — the
  downstream consumer

## History

**Materials-response supersession (`feature/materials-response-build`).**
The `inflict` producer was upgraded by the follow-on materials-response
build: what this doc's earlier text called "magnitude-only severity + a
bijective `mechanismToType` switch + a binary `isSiteCovered` gate" is
retired. `inflict` now resolves a `Channel` insult outside-in through the
covering stack into the tissue — **both** severity AND trauma type come from
`MaterialApi`'s response function, and coverage is a *degree* (the stack),
not a presence flag. The mechanism vocab unified into the channel set
(`thermal`/`tearing` keep a magnitude-only passthrough); `TraumaType` grew
`puncture`. See [materials-response.md](./materials-response.md). The
sections above are current; this note marks the seam for readers of the
retired harm plan.

---

The original build shipped in the harm-driver branch (the `feat(harm): Phase
0–6` commit range through the pre-merge sweep). Two design→implementation
shifts landed during MR review and are worth noting because the retired
plan/requirements docs describe the pre-review shape:

- **`HarmApi` → `ConditionApi`.** The producer Api was promoted from a
  harm-only surface to the gated facade over the whole vitals *condition*
  surface — `inflict` (trauma) plus `afflict` / `relieve` / `conditionsOf`
  (afflictions). Reserves (endurance) and transient combat flags are
  distinct axes and stay out. Logic singleton at `/obj/api/condition`.
- **Push tick → reconcile-on-read.** Wound progression was originally
  planned as a `ScheduleApi.recurring` push with an in-memory tick-handle
  map re-armed on hydrate from `Avatar.enter` + `NPC.postRegister`. It was
  reworked to reconcile-on-read on the `VitalsMixin` read path (a
  persisted per-trauma `tickedAt`), which deleted the re-arm seam entirely
  and made harm consistent with the metabolism / thermal reconcile
  drivers. The demonstrator room's in-world seed exit (first off Dave's
  Bar, then the Terminus Terminal hall) was retired at the sweep — every
  real content-area host broke a standup/fast-travel invariant — so
  `GlassAlley` ships as a class + integration fixture, its reachable-in-
  world placement deferred.

## Stabilization (shipped 2026-07-31)

`treat` now does two jobs — see [mortality.md](./mortality.md). Beyond
dressing a wound, it pulls a body out of the **dying window**:

- a dying body is treatable with **no wound to dress** (cold or a toxin
  leaves nothing to bandage);
- the graded outcome gates the rescue — a failure spends the dressing
  without holding them;
- **rescued is not healed**: `stabilize()` drops the dying record and
  touches nothing else, so a body still under its threshold falls back into
  the window on the next reconcile.

`assess` reports the remaining window, sharpened by competence — the
information rule, not an outcome rule.

`ConditionApi` grew **`die`**, the single death transition every lethal
driver now reaches (replacing seven scattered sites), plus
`embodyForSession` / `reembody` for the recovery arc. The per-driver dying
windows live with the physics that justifies them, in each driver's own
`*_DEFAULTS`.
