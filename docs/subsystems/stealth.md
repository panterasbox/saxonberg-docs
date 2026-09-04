# Stealth & deployables (the actor-face of concealment)

The **player-facing half of concealment**: hiding *yourself*, moving under
cover, ambushing from concealment, and planting *your own* traps —
completing the vertical [concealment.md](./concealment.md) opened (that
build made concealment *authored-static*; this one makes it
*actor-driven*). It leans end-to-end on shipped substrate — the per-viewer
detection engine (`PerceptionApi.perceives`), the `ConcealableMixin` gate
(now given a *derived* level for actors), the hazard `Trap`, the
behavior/brain system, the parcel property gate — and the unified
[accountability.md](./accountability.md) ledger so ambush and traps derive
`crime` the same way combat does.

## `hide` — a Character-level actor state, level derived on the engine

**`HidingMixin`** (`lib/concealment/Hiding.ts`, `Mixins.Hiding`) is composed
on `Character` **outside** `Creature`'s `ConcealableMixin` (the seam that
makes the override reach). It holds two persistent fields — `hiding:
boolean`, `hiddenLevel: ConcealmentLevel` (the `PosedMixin` precedent) — and
**overrides `getConcealment()`**: return `hiddenLevel` while `hiding`, else
`super` (the authored static field — an NPC authored lurking keeps its
band). So the shipped `perceives` gate, the honest-fog enumeration seams,
and `search` all resolve an actively-hiding actor **for free** — a hidden
actor is a concealable like any other.

**The level derivation is a rule → it lives on the gated engine.**
`PerceptionApi.hideLevelFor(actor, stealthBand)` (a sibling of
`effectivePerception` / `modeAttention` on `PerceptionLogic`) is **pure and
deterministic** — a weighted score of `stealth` competence × available room
cover × darkness × stillness (a low posture), mapped to a
`ConcealmentLevel` by the `stealth.hide.band.*` thresholds (a score below
`band.subtle` fails to conceal → `obvious`). **Snapshot at entry:** the
`hide` verb `await`s `competenceBandFor(actor, 'stealth')`, calls
`hideLevelFor`, and stashes the band into `hiddenLevel` (the combat
"snapshot competence at open" precedent); `getConcealment()` then reads the
snapshot synchronously — **no async at the perceive gate**. This keeps the
gate sync + deterministic and puts the level math on the engine that owns
detection (no parallel `StealthApi`, mirroring "no `DetectionApi`").

**Discovery is non-sticky while hiding.** `HidingMixin.getDiscoveryKey()`
returns `undefined` while `hiding`, so a searcher who beats the level
reveals the actor **in the moment** (the `resolveSearch` found list) but
records no permanent per-viewer `DISCOVERY` — **re-hiding is a fresh
contest**, not "found forever." (A non-hiding NPC's authored concealment
keeps its durable `templatePath` key via `super`.)

**Break conditions, each at its natural hook:**
- *Motion* — resolved at `Mobile.traverse` with the mode in hand (below).
- *Attacking* — `AttackController` clears the attacker's `hiding` (you
  reveal yourself by striking), reading it *first* for the ambush check.
- *Search* — per-viewer, free (`resolveSearch` reveals a hider it beats).
- *`unhide`* — the deliberate drop.

**Verbs:** `hide` / `unhide` (thin controllers, narration on the
controller). Category `perception` — hiding is the actor-face of the
concealment contest, the sibling of `search`.

## Motion degrades concealment (observer-side care↔speed)

The shipped `movement.attention.*` dials are the *self-side* of the
care↔speed axis (how a mode changes what **you** notice). This build lights
their **observer-side**: `PerceptionApi.motionExposure(mode)` (the mirror of
`modeAttention`) returns the concealment **bands** a move at that mode strips
from a hiding mover, applied at `Mobile.traverse` via
`HidingMixin.degradeHide`: `sneak` holds (0), `walk` degrades one band, `run`
clears hiding outright (a hidden mover you `run` past is fully exposed). The
degrade is persistent (until you re-hide), so the shipped perceive gate
resolves the newly-exposed mover per-observer for free.
`movement.concealment.{sneak,walk,run}` dials.

## Ambush denies the poise contest

`CombatOpenOptions.ambush?: boolean`. `AttackController` resolves it — the
attacker is `hiding` **and** the defender does not perceive them
(`!perceives(defender, attacker)`, the defender's awareness warmed first) →
`ambush: true` — then reveals the attacker. `openSessionImpl` consumes it:
when ambush, the defender's `Poise` is eroded from full at open so the
crossing arms a live opening (`combat.ambush.poisePenalty`). Surprise
**denies** the opening poise contest — it is *not* a damage multiplier; the
exchange still routes through `ConditionApi.inflict`, and the "crit" is the
earned open window. An ambush **kill** under imposed lethal terms derives
`crime` through the same accountability ledger. See [combat.md](./combat.md).

## The `wary` brain — detect → react

`lib/behavior/wary.ts` (a canned brain, the `greets`/`introduces`
precedent). On a presence-gated cadence, it scans the creatures sharing its
room, runs the shipped `perceives` with a config-driven `alertness`
attention, and on a **state-change to detected** (a per-`state` seen-set
delta) reacts — a spoken challenge, an alert emote, and optionally opening
combat (`config.hostile`). On a failed detection it stays oblivious: a
good-enough hider (concealment requirement > alertness) walks past unseen; a
visible intruder is always caught. Detection is free (the NPC is already a
valid `perceives` viewer); the only new surface is behavior. See
[behavior.md](./behavior.md).

## Player-trapper — `TrapKit` + `arm`

**`TrapKit`** (`lib/hazard/TrapKit.ts`) is a thin carried `Thing` (obtained
ready-made — **no recipe/component economy in v1**) holding a `trapTemplate`
(a `/stuff/thing/traps/` generic). **`arm <kit>`** (`device` category, the `disarm`
sibling) clones that generic, sets the deployed trap's concealment from the
placer's own `stealth` via `hideLevelFor` (the shared spine with
self-hiding), stamps the placer on it (`HazardMixin.placedBy`), places it,
and spends the kit (one trap per kit).

**Anti-grief ships inline** (a completeness requirement, not a follow-up):
- the **property gate** — `arm` is free on ground you hold (the shipped
  `AccessApi.can` write test) or on public/`core` ground; **refused** on
  another owner's property (the parcel substrate);
- **crime-marking is inherent** — the deployed trap's `placedBy` makes a
  spring on a non-consenting sentient append a `harm` accountability row
  (at `HazardMixin.deliverHarm`), from which `crime` derives — no flag.
- **pick-up-your-own** — a placer may `get` their un-sprung placed trap
  (identity check on `placedBy` in `GetController`); a non-placer is
  refused (and a concealed one they can't perceive never resolves anyway).

## The `stealth` Discipline

`seeds/lib/advancement/Discipline/stealth.yaml` — the opposed sibling of
`awareness` (one contest, two competences): `stealth` grades a hider's
`hideLevelFor` level and a placed trap's concealment; `awareness` grades the
watcher who tries to see through them. Pure data, `channel: skill`, **no
`conferrals`** (`hide`/`arm` are universally afforded — competence only
grades). See [advancement.md](./advancement.md).

## The demonstrator

A **watchpost** in the newbie-wilds locality (`domain/newbie-wilds/`),
reached by one plain ground exit off the treeline (which is spawn-override
reached and stood up by **no** test — so the room + sentry pull into no
standup/fast-travel cascade; the GlassAlley failure, avoided by
construction; it asserts no exit count, composes no `FastTravelMixin`,
appears in no TPA manifest). A **`wary` sentry** (`alertness: 3` — a decent
hide slips past, a poor one is hailed) keeps the ground; a dropped
**`TrapKit`** waits in the bracken to `arm` on this public ground for the
next traveller to spring.

## AppSettings dials

`stealth.hide.competencePerBand` / `coverWeight` / `lightWeight` /
`stillnessBonus` / `band.{subtle,hidden,deep,buried}` (the hider's derived
level); `movement.concealment.{sneak,walk,run}` (observer-side
motion-degrade); `combat.ambush.poisePenalty`. Each read with a
seeded-literal fallback (the `dialNumber` idiom).

## Deferred (named)

The knowledge economy (sharing/selling found secrets, maps as currency);
ranged/remote/linked traps + resettable/rearming placed traps; non-combat
threat-reads (`assess <person>` out of combat); disguise-as-concealment
unification (identity stays belief's axis).
