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
the gated `ConditionLogic` singleton at `/platform/idea/api/condition`). Reserves
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
energy — vs the corrosion variant `{ mechanism:'corrosion', site, energy,
corrosiveTo }`, which carries the **agent's chemistry** (the material tags
it eats through) because an energy scalar cannot.

⭐ **The corrosion producer is a substance in contact, not an energy
channel.** A caustic material delivers through
`Material.corrodeOnContact(victim, {energy, site})` — it reads its own
`corrosiveTo` and forwards this variant through the door — which is what
makes a thrown flask, a conjured acid (the acid-splash spell), and a
spilled vial all corrode with no bespoke path. The lime-seep hazard is the
original producer; see [materials-response.md](./materials-response.md) §
*How corrosion reaches a body*.

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
  ⭐ The affliction twin: **`AfflictionRecord.inflictedBy`** (the
  consequence build, W7) is stamped from the same
  `getActingAuthor()` at the `VitalsMixin.afflict()` door, never
  overwriting a stamp a producer set deliberately. The gap it closed: a
  wound had always known who dealt it; an affliction never did — so
  poisoning, the one kind of harm that is deliberate, premeditated and
  quiet, was the one kind the world couldn't attribute. An unattributed
  affliction stays unattributed on purpose (most harm has no author;
  inventing one would sweep the weather into the crime ledger).
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

## The ten trauma behaviors

Live in `platform/idea/Condition.ts`, co-located with the `Trauma` value in
the closed `TRAUMA_BEHAVIOR: Record<TraumaType, TraumaBehavior>` roster.
The tenth is **`foreign-body`** (clinical-medicine D6) — a penetrating
wound with the thing still in it (`Trauma.foreignBody`, prose). It bleeds
at `FOREIGN_BODY_BLEED_SCALE` (0.35×) of the open rate while embedded (the
object tamponades its own channel), never knits until extracted, and only
`resolution: 'extraction'` (the `operate` catalogue) resolves it — after
which it is a dressed puncture. It joins `BLEED_FAMILY` (the open-wound
sepsis clock is its deadline). Producers opt in via
`EnergyInflictSpec.embeds` / `HazardDeliveryOptions.embeds`; a `puncture`
at/above `EMBED_MIN_SEVERITY` carrying `embeds` mints it.
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
- **puncture** — delegates `onset` / `tick` / `resolve` / `reopen`
  **wholesale to `LACERATION_BEHAVIOR`**, differing only in `describe`
  ("a bleeding puncture wound of …"). ⭐ That is the honest model, not a
  shortcut: a stab and a slash both open a vessel and both answer to
  pressure and a dressing. The two exist as separate `TraumaType`s
  because the **channel** that produced them differs (`point` vs `edge`)
  and because a treatment build will eventually want to tell them apart —
  a deep narrow tract and a long shallow one are not the same surgical
  problem — not because they behave differently today.
- **contusion** — mild, self-resolving severity decay; no bleed.
- **fracture** — a slow natural heal. **The impairment is a derived read**,
  not a tick effect: `Vitals.isSlotImpairedByTrauma(slot)` folds into
  `Slotted.canOccupy` alongside the anatomy gate, so a fracture at a slot's
  `bodyPart` above `FRACTURE_IMPAIR_SEVERITY` greys that slot's
  affordances — and healing/clearing the fracture restores them with no
  separate un-impair step.
- **burn** — real behavior: severity + a slow heal at its own rate.
- **avulsion** — behaves as a **severe laceration** (floors severity via
  `AVULSION_SEVERITY_FLOOR`, bleeds, shares the clot gate) and, at or past
  `HARM_DEFAULTS.SEVER_SEVERITY`, **takes the part off** — see *The sever*
  below. Reachable from a weapon since the injury build: the `edge`
  channel climbs a ladder (`response.edge.avulsionThreshold`) exactly as
  `blunt` climbs to `fracture`. Before that it arrived only through the
  `'tearing'` passthrough, which nothing in the game produced.
- **rupture** *(interior)* — a **torn organ**, and the first wound in the
  game you cannot treat. Laceration's bleed with `resolve` a no-op (you
  cannot put pressure on a liver) and `resolution: 'surgery'`, a token
  nothing offers — `mismatchLine` renders it as *"It wants surgery."*, so
  the game says exactly what is wrong and exactly why the bandage is no
  use. Minted by the depth ladder on a `blunt` channel past
  `response.blunt.ruptureThreshold`.
- **frostbite** *(reserved — the cold channel)* and **caustic**
  *(reserved — the corrosion channel)*: see materials-response.md.

## ⭐⭐ The sever — losing a part

`BodyPartDelta.missing` shipped persistent, cascaded through
`isSlotDisabledByAnatomy`, and **was written by nothing**. Its one writer
is now `VitalsMixin.severPart(key)`, called from `AVULSION_BEHAVIOR.onset`
when **three** things are true: the wound is at or past
`HARM_DEFAULTS.SEVER_SEVERITY`; the body plan marks the part `severable`
(that field's first production reader — you cannot lop off somebody's
chest); and the blow was **authorized to maim** (`Trauma.maimAllowed !==
false`).

⭐⭐ **Maiming respects the fight's terms.** `maimAllowed` is undefined
for every environmental source — a fall onto spikes, a hazard, a beast's
cull all take a limb if severe enough, because nature does not ask
consent. The one producer that suppresses it is **combat between
sentients**, which sets `maim: false` when the session's terms are not
`lethal`: a sparring bout leaves a severe avulsion, never a severed part.
This is the same consent structure the two-stage defeat/coup enforces for
killing — a maiming is at least as grave as a kill, and must not fall out
of a non-lethal exchange.

Three things happen, in order:

1. **The subtree goes, not the part.** Severing `body.arm.left` marks
   `body.arm.left.hand` missing too — a hand with no arm is not a thing a
   body can have. Transitive over `BodyPart.parent`.
2. **What the part held falls.** Every slot whose `SlotSpec.bodyPart` lies
   in the severed subtree is vacated and its occupants moved to wherever
   the body is. ⚠ `Slotted.canOccupy` only ever refused *new* occupancy of
   a missing part's slots — a severed hand kept its sword.
3. **It persists for free.** `bodyPartDeltas` is already
   `{persistent, runtimeState}`, so the loss rides `PersistableApi.capture`
   through a logout and into a corpse. No new storage, no migration.

⚠⚠ **`onset` now runs AFTER `afflict`**, at all three terminal paths. The
old order was harmless only while every `onset` mutated the trauma *value*;
it stopped being harmless the moment one could act on the *body*. A wound
a conferred immunity refused must not take an arm with it.

### ⭐⭐ A missing vital organ is lethal — the anatomy death floor

Losing a part that **governs** a life-critical capacity (consciousness,
circulation, respiration) begins the dying clock:
`reconcileConditions` calls `beginDying('decerebration',
VITAL_ORGAN_LOSS_DYING_WINDOW_SEC)` when `hasMissingVitalGovernor()`. A
severed head is the reachable case today — brain gone means no breathing
drive and no airway, not merely unconscious, which was the shipped answer
(it left a decapitated body beating away forever).

⚠ It sits **above the all-empty guard** — a severed part writes no vital
sign and its wound may have clotted to nothing, so a body whose only
problem is a missing head would otherwise reach the guard, find no active
condition, and return whole-signed and immortal (the same trap the bleed
floor fell into pre-W-A4). Gated on `bodyPartDeltas` being non-empty first,
so an untouched body — almost every body, almost every read — pays a single
map-size check.

⚠⚠ **Death clears the anatomy** (`resetAnatomyToSpeciesBaseline`, in
`divideBody` beside the vital reset). Without it a decapitated player
reembodies headless and the floor re-kills them on the first read — dead
on arrival, forever. A body that comes back from the passage comes back
**whole**, exactly as it comes back with full blood and no conditions;
resurrection restores the body. A **living** limb-restore — a surgeon, a
shrine, a prosthetist — is a separate content-facing path, deliberately
unbuilt (see below).

### ⚠⚠ What this is NOT, and what the next build owes it

Severing shipped as **substrate, not gameplay** — the mechanic is sound
and the persistence is right, but the design space around it was never
worked. What exists: a severe enough authorized edge blow to a severable
part takes it; the function axis reads the loss; it is lethal for the
head, survivable for a limb; death restores it. What does **not** exist,
and is the sever-gameplay slate's charter:

- **Reachability.** Combat's `siteFor` returns torso/head only, so the
  one severable outcome a fight can reach today is **decapitation**. A
  hand or a leg cannot be targeted — the aim-derived called shot
  (`combat-slate`) is its prerequisite. Until then, limb loss is
  effectively hazard-and-cull only.
- **The living way back.** A content-facing `restorePart(part)` a temple,
  clinic or prosthetist calls on its own terms — the `reembody` shape, on
  anatomy. The engine does the part; the world decides the cost and who
  can. Rare by construction, because someone must author the place.
- **What a stump MEANS.** A one-handed character's economy, the prosthetic
  as an augment that re-enables a slot (`augmentation-slate` already has
  the shape), whether an NPC ever comes at you already maimed.

→ `physiology-slate § Part 7h`.

## ⭐⭐ The depth ladder — what a blow reaches under the skin

A wound past `response.depth.reachThreshold` has excess severity left
over, and the excess reaches the **interior** parts sitting under the site
(`BodyPlan.interiorChildrenOf`), **largest cross-section first** — a
bigger organ presents more of itself to whatever is coming through. Each
takes `response.depth.stepPerOrgan` out of what remains, so a deeper blow
reaches **more** organs rather than merely hurting the first one worse.

The channel decides what it does to the organ: `point` → puncture, `edge`
→ laceration, `blunt` → **rupture** past `response.blunt.ruptureThreshold`
else contusion (a concussion; a bruised liver). Each interior trauma goes
through the same `afflict` door and is separately vetoable; they come back
on `InflictOutcome.reached`.

⚠⚠ **No roll anywhere.** The biggest organ under a site is hit first,
every time; a deeper wound reaches further, every time; a student can
derive both from `assess`. A surface-fraction-*weighted* site pick would
be a roll deciding *what your action did*, which `docs/uncertainty.md`
bans outright. The honest improvement is aim-derived and deterministic —
a called shot — and that is a combat build.

**Interior wounds are invisible and undressable.** `assess` gates naming
on the observer's *real* medicine competence (not the self→expert
shortcut — what your own liver is doing is not visible to you *because* it
is you), so an ordinary player reads *"Something is wrong inside; you
cannot tell what"* and learns more from a competent stranger than from
looking. `treat` refuses a dressing on an interior site — and the gate is
on the **site**, not the trauma type, because an interior *puncture*
resolves by `dressing` like any other bleed.

## ⭐⭐ The function axis — a wound costs a CAPACITY

The build's centre. A wound stops costing a number and starts costing
grip, gait or consciousness.

- **`VitalsMixin.functionAt(part)`** — `min` over the part's own tissue
  (`1 − Σ severity × lossPerSeverity`) and everything **upstream** of it:
  the limb it hangs off, the nerve that carries it, the vessel that feeds
  it, and recursively whatever reaches those. Banded
  `full · impaired · failing · lost` (`lib/vitals/BodyCapacity.ts`).
- **`VitalsMixin.capacity(k)`** — `governs` combines by **min** (one
  brain: lose it, lose the capacity), `serves` by **mean** (two legs: one
  gone is a hobble, not a halt). A capacity nothing governs or serves
  reads `full` — a body that never had hands has no `manipulation` to
  lose, and that is data, not a guard.
- ⭐ **Quadriplegia and paraplegia fall out for free**: the lower spine
  hangs off the upper, so a high cut takes the arms *and* the legs and a
  low cut takes only the legs. Two `parent` edges and a `min`.

⚠ The recursion is load-bearing. The **arm** names the spine in
`innervatedBy` and the hand names nothing, so a one-level walk would leave
a severed spine with a happily gripping hand — and it is what lets
innervation be authored only where the supply path *diverges* from the
tree.

**Three consumers, no new guards.** `isSlotImpairedByCondition` is
`!canGrip(slot)` and nothing else (folding in the anatomy gate, and —
unlike the old exact site match — a wound on the *arm* now refuses the
*hand's* slot); `getConsciousness` reads the `consciousness` capacity when
the plan authors a governor for it; `drainForLimp` is
`1 − capacityScalar('locomotion')`, which finally catches a fractured leg,
a missing leg, and a spine wound — all three invisible to the old
`body.leg*` string sum.

## ⭐⭐ Circulation — shock before death

Blood pressure is **derived from circulating volume** on every reconcile,
and the shape of the curve is the teaching:

- **Systolic holds, then falls.** Nothing moves until
  `SHOCK_COMPENSATED_LOSS` (15 %) is gone. That plateau is ATLS class II,
  and it is the single most important fact about haemorrhage: *a patient
  can be seriously bled with a normal blood pressure right up until they
  are not.*
- **Diastolic rises first.** Vasoconstriction pushes it up while the
  systolic holds, so the **pulse pressure narrows** — the earliest sign
  there is. Past the plateau both fall together.
- **`hypovolemic-shock`** spawns at 30 % loss, relieves at 25 %. The dying
  window opens at 36 %, so shock always precedes death by a real interval.

⚠ The derive runs **above** the clock guard in `reconcileConditions`,
beside the burden law and for the same reason: it is a live read of
present volume, not an integration over elapsed time. It arms
`_reconcilingConditions` by hand, because that flag is not set until far
below and this derive *writes* where the burden law only reads.

⚠⚠ It does **not** write `heartRate` — the sustained-shock arm owns that
sign, and two writers on one vital is the defect `lint:condition-arms`
exists to prevent. Heart-rate compensation (the tachycardia that precedes
the pressure drop) is a real sign and a real gap → `blood-slate`.

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

## ⭐⭐ The effect channel — what a condition DOES

`Condition.signature` shipped as `{sign, delta}[]`, persistent,
authorable, spoiler-levelled, with a public accessor — **and nothing
anywhere read it.** All twenty-three shipped rows author `signature: []`
because there was nothing else to author. The one effect any affliction
had on a body was a hydration drain **hard-coded inside**
`Vitals.progressInfection`, for pathogens only, that no row asked for and
no row could ask for.

`VitalEffect` is now a four-kind union, interpreted by one method,
`Vitals.applyEffects(effects, intensity, elapsedSec)`, which every arm
calls after advancing its law:

| kind | what it does | when |
|---|---|---|
| `vital` | integrates `perHour` on a vital sign | per reconcile |
| `reserve` | integrates `pctPerHour` on a biological reserve | per reconcile |
| `function` | what the wound costs the PART it sits on, per unit of severity — the function axis reads it | read |
| `expression` | competence suppression in bands | read |

⚠ **`delta` became `perHour` deliberately.** A raw delta has no answer to
*"applied how often?"*, so it could only ever have meant "once, on some
tick nobody defined" — which is why it was never wired. A **rate**
integrates over whatever elapsed, which is the only shape that works with
reconcile-on-read and an absent player.

`intensity` is the condition's own severity axis (a stage, a wound
severity, a pathogen load), so one row's declared effect scales with how
bad *that* condition has got and the arm that advanced it never has to
know what the effect was. `TraumaBehavior` carries the same `signature`
for the Kind-B half, declared on the closed engine table beside each
type's `tick`.

⚠⚠ **A `vital` effect naming a sign the body does not have is a
deliberate silent no-op.** The `constructa`, `plantae` and `fungi` clades
exist; a construct that takes an edge blow has a wound and no bleed, and
that is the honest answer. A species says so by authoring the band's
`baseline` at zero (`Vitals.hasVitalSign`). It has a **test**, because
the failure mode is the silent-and-closed one: an effect that does
nothing because nobody wrote the branch reads identical to one that does
nothing because the author said so. `lint:conditions` is the other half
of that guard.

## ⭐⭐ The progression laws — seven arms become five

`reconcileConditions` grew seven arms, each added by a different build,
each discriminated by **which optional field happened to be set on the
record**: `magicOrigin` → decay, `pathogenLoad` → logistic, neither →
dwell. The *shape of the record* decided the law, so a row could not
choose one and every new law meant a new arm. An eighth mechanism
(`Metabolic.reconcileToxinConditions`) kept its state outside the
collection entirely and mirrored a band into `stage`.

⚠ The trap sprang once already, with a comment proving it: the arm that
filled `ProgressionSpec` recorded that the field *"was authored by three
rows, and was read by nothing"* — and **added an arm** rather than asking
why. `lint:condition-arms` was written before this build for that reason.

A row now declares `progression.law`, one of **`stage` · `decay` ·
`logistic` · `burden`**, and one `afflictions` arm dispatches on it. The
three affliction arms collapse to one (**7 → 5**), and
`reconcileToxinConditions` stops writing `stage` (**parallel 1 → 0**).
The rule the ratchet enforces: **a condition's progression law and its
effect on the body are independent — a new condition kind is a law plus a
signature, never a new arm.**

⭐ `progression: null` is a first-class answer, and five shipped rows use
it: a driver outside the condition collection owns their clock (the
metabolic collapse gate, respiration's `spo2`, thermal's temperature).

⚠⚠ **The `burden` law runs above the world-clock guard**, and that is
load-bearing rather than an optimisation: a toxin's stage is a live
*read* of how much of it the body is carrying, not a counter that
accumulates. Behind the clock guard — or behind the presence-freeze
guards — a body that had just drunk would read sober until enough
game-time passed, which is both wrong and a regression against the
parallel store it replaced (that derived synchronously). Everything else
in `reconcileConditions` is about integrating elapsed time; this is not.

## ⭐⭐ `resolution.by` — the treatment matches the condition

`Condition.resolution.by` shipped authored on two rows and read by
nothing, so **every treatment was the same treatment**: `treat` picked the
worst wound and applied whatever was to hand, and a bandage on a burn
worked exactly as well as water on it.

Each condition now declares what relieves it — a `Condition` row through
`resolution.by`, a trauma type through `TraumaBehavior.resolution` — and
`treat` resolves what the medic is **offering** and matches it:

| offered | is | treats |
|---|---|---|
| a `Dressing` item | `dressing` | laceration · puncture · avulsion |
| a `Bulkable` vessel with something in it | `fluid` | burn |
| nothing — the medic's own hands | `medicine` | an illness (the load knock) |
| — | `rest` | contusion · fracture · `recovering` |

⚠ **A mismatch is refused with prose that names what the wound wants**,
rather than silently doing the wrong thing. *That refusal is the
teaching*: you learn a burn needs fluid by being told a bandage does
nothing for it. `treat <target> with <item>` chooses explicitly.

⭐ Fluid is **drunk**, through the shipped `Metabolic.ingest` path, so
water is a real supply that runs out — a medic with an empty skin has
nothing to give, which is what makes a priced treatment mean anything. A
dressing is spent; a vessel is only emptied.

⭐ **Bare hands are a treatment, not an absence**, and that is what
finally reaches `tendInfection` — a complete, commented private method
that had been in `TreatController` since it was written **with no caller
anywhere**. The `mustHaveDressing` validator is gone with it: it would
have refused a medic carrying water.

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
  DressingMixin(Thing)` (`obj/Bandage.ts`, the `Coin`
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
  `ActSignature` (`creditDeed`) into the treater's
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
  ⭐ **`analyze patient`** is the trade's diagnostic surface above
  `assess`, shipped by the `trade-medicine` pack (the stanza is on the
  platform's `analyze` view, the controller is the pack's — the
  instrumentation split). Its line: **competence buys what you can SEE,
  never what you can DO** — an untrained eye knows something is wrong; a
  novice reads the signs; a competent medic knows which conditions could
  produce them, *plural and unranked* (that ambiguity is what makes the
  choice a choice); proficient knows what would treat it and how it
  spreads; expert reads how far it has gone. Nothing on the ladder makes a
  treatment work better. The forensic twin, `analyze postmortem`, is
  [mortality.md](./mortality.md).
  **Why the diagnosis loop is a trainer, not a script.** Three shipped
  decisions combine: honest opacity (no gauges — you read the world, not a
  stat); every `Condition` carries prose `observableSigns`; and the
  catalogue's two dozen rows have *overlapping* signs. So **differential
  diagnosis emerges rather than being scripted** — flushed, sweating and
  disoriented is consistent with hyperthermia, with a toxin burden and
  with an infection, and separating them takes more signs, a history or an
  instrument. That is what scripted clinical sims structurally cannot do:
  their scenarios have one correct answer fixed in advance, while this
  patient's state derives from a model, so the reasoning is real even when
  the case is unremarkable. The anti-wiki rule carries over from farming
  verbatim: knowing a condition's sign set never tells you that *this
  patient* has it — knowledge is portable, the assessment is not
  skippable. *(Graduated from the health-vertical slate, 2026-09.)*
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

## ⭐⭐ Recovery — care buys RATE (the recovery build)

The harm engine healed on read at a flat per-type rate; recovery makes the
rate something **care** raises. The keystone is one number.

### The `mend` split, and the convalescence factor `k`

`TraumaBehavior` gained **`mend(host, t, elapsedSec, k)`** — the HEALING
half, split out of `tick`. `tick` now carries only what HARMS and cannot
be sped up (the bleed drain, the caustic's growth, the burn's weep via
`signature`); `mend` is the severity decay, and the ONLY half `k` scales.

`Vitals.convalescenceFactor()` computes `k` once per reconcile:

```
k = POSTURE_REST_BASE[posture] × surface.restQuality × surface.convalescence
    × (1 + carerBonus) × Π conditionFactors        (floored at CONVALESCENCE_FLOOR)
```

A **bed, a carer and a spell are three payers of the one number**. The
posture→rest table lives on the posture vocabulary (`POSTURE_REST_BASE` in
`lib/character/Posed.ts`), read by BOTH this and metabolism's stamina
recovery, so the two drivers never disagree on which surface a body is on.
`PosturedMixin.convalescence` is a SECOND field beside `restQuality`: a
clinic cot mends a wound faster (`convalescence 2.0`) without being a
better night's sleep than a four-poster (whose `restQuality` alone drives
stamina). `assess` says *"mending well / steadily / slowly"* from `k`.

⭐ **D3a — convalescence requires SAFETY.** `k` is **0** (overriding the
floor) whenever the body is in a live `CombatSession` or was harmed within
`CONVALESCENCE_SAFE_DELAY` (a transient `_lastHarmedAt`, stamped in
`afflict` for trauma/shock). This is the **intent-agnostic** answer to
combat-logging: intent is undetectable, so we gate on the *situation*,
identically online, linkdead or logged off. See the
absent-body slate.

### The offline carve

`Trauma` carries TWO stamps. **Both** freeze on linkdead — the broad "a
linkdead body integrates nothing" invariant every other arm (dying,
electricity) keeps — but they differ on the FAR-PAST gap: the **harm arm**
(`tickedAt`) drops it (*being away must never bleed you*), the **mend arm**
(`mendedAt`) INTEGRATES it (*being away must never cost you, and mending is
never a cost*). So offline mend is the LOGGED-OFF reconnect: an evicted
body is not reconciled while away, so on return (no longer linkdead) its
big gap integrates in one read. Both arms live inside the ONE trauma
`for…of`, so `lint:condition-arms` still counts a single arm. This IS the
shipped sleep-as-logout mechanism given a convalescence purpose; the mend
depends on the body re-occupying its rest surface (a full snapshot restore
degrades to floor rate — a shared metabolism+vitals seam on `Posed`'s
restore path).

### Every wound treatable

`resolve` is now live for the mechanical types too: a fracture is **set**
(`resolution: setting`, a splint), a rupture is **operated** on
(`resolution: surgery`), a burn is **cooled**, a frostbite is **rewarmed**
— each reads *set / closed / cooled / rewarmed* and knits at a faster
treated rate, graded by `Trauma.careQuality`. `Vitals.applyTreatment(wound,
{by, efficacy, treater})` is the ONE primitive every consumer calls
(`TreatController`, `OrderController.treatWorst`, the instruments, the
nurse); it runs `resolve`, stamps `careQuality`, and seeds infection.

The verbs, and what affords each: **`treat`/`undress`/`dose`/`tend`** on
`VitalsMixin.self` (the body affords its own first aid); **`cool`** on
`WaterFixture.peers` — and bare **`wash`** (no object) washes your hands
there, the hygiene half folded into the platform `wash` verb rather than a
separate `scrub` (hands have no object arg, so nothing was widened; `rinse`
takes a body ARG and stays separate); **`warm`** on `FurnaceMixin.peers`;
**`splint`/`operate`** on the trade's `Splint`/`SurgicalKit` instruments
(`trade-medicine`). `dose` reads an `antidote:<toxin>` **Material tag** off
a vial and crashes the matching burden (D7 — an antidote is a substance
with a tag, no new mixin). ⚠ The bone-setting verb is `splint`, not
`set` (`set` is a scripting builtin).

⭐ **`operate` is the honest minimum, not the craft.** It stops a rupture's
cavity bleed in one decisive FIELD act — deliberately shallow (instant,
single-wound, cannot-fail: an untrained hand is refused, not risked).
*Surgery the practice* — a durative, interruptible operation with a
bleed-during clock, a surgical team, a theatre, and anaesthesia as the
epoch dial, and the install act for augments/prosthetics — is its own
build (clinical-medicine-slate); this verb is
the stub that vertical will wrap, not replace.

### Hygiene and the festering wound

`HygieneMixin` (on `Creature`) is one `washedAt` stamp with a derived
`handsCleanliness()` that decays over `HYGIENE_SOIL_SEC`; `scrub()` cleans
(bare `wash` at water drives it), `soil()` dirties (treating a bleed soils
the treater). **Wound sepsis**
(`Condition/pathogen/wound-sepsis`, `reach: infect`) is the shipped
in-host infection arm with a new SOURCE: `applyTreatment` seeds it when a
bleed-family wound is dressed with dirty hands or poor care, and the harm
arm seeds a wound left open past `SEPSIS_OPEN_ONSET_SEC`. Its severity
drains hydration into the shipped dehydration → dying cascade — nothing
new kills anyone. `assess` reads *festering*.

### The carer, and the alarm

A carer is a **`TendingEngagement`** on the carer's `attention` slot
(one patient at a time), linking `patient._setCarer(carer, band)`; the
patient's `k` reads the carer's medicine band as a bonus while the carer is
present, conscious, and still holding the engagement. Aldis Verrow's
`nurses` brain (`trade-medicine`) triages the room and dresses or tends,
deterministically, from finite supply.

⭐⭐ **The notify layer is an ALARM, not a heartbeat.** Recovery advances
silently under derive-on-read; to keep the infection DEADLINE visible,
`Vitals.nextInterestingAt()` (a pure read) gives the soonest pending
transition, and a **one-shot** `ScheduleApi.schedule` is booked at it,
canceled and rebooked on state change. ⚠ NEVER `recurring`, never a
cadence, never a per-body sweep — a healthy body books nothing. The
callback only calls `reconcileConditions` + a message push; **correctness
is independent of it firing** (death itself is derive-on-read — the alarm
buys TIMELINESS, never VALIDITY). *Delete the scheduler and the game is
still correct, only less timely.*

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
  distinct axes and stay out. Logic singleton at `/platform/idea/api/condition`.
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
