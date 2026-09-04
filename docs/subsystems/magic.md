# Magic — the effect substrate + casting core

The first magic build: **magic = spend a reserve → fire a list of
declarative effects → they land in the systems we already have.** Magic
is a new *trigger*, never a new *mechanism* — the payoff of the
frontier-physics arc (electricity → storms → fire) built expressly so
magic would have honest channels to actuate. A firebolt injects real
heat and real combustion takes over ([fire.md](./fire.md)); a spark
imposes a real potential and the real conduction graph resolves it —
including the caster standing in the same pool
([electricity.md](./electricity.md)); fear is a real condition gated by
the target's live Composure.

> **Authoring magic content? Read
> [docs/arcane-science.md](../arcane-science.md) first.** It carries
> the fiction's physics — the one postulate, the laws, the price
> list — and **8 binding content-authoring rules** that govern what a
> spell may cost and what an effect may claim (costs are energy
> committed in τ (`1 τ ≡ 1 kJ`), derivable from the price list; effects author
> energy *delivered*, never outcomes; a new noun needs a real physical
> register; momentum is conserved; if it can't be dimensionally
> analyzed it doesn't ship). Those rules exist so the magic system can
> be *taught as a college course* — a spell authored off the price
> list silently makes the curriculum a lie.

Design source: `docs/slates/deferred-rpg/capability-magic-slate.md`
Part IV (the locked model). Consumers this build deliberately does NOT
include: the inquiry substrate (discovery/publishing), the magic-items
tier (Consumable/BUC), Transform/polymorph — each its own later build
over this same substrate.

## ⭐ The computed cost (`costModel`)

A spell's authored `cost` is a flat number, and for twelve of the
thirteen shipped spells that is the whole price. `teleport` is the
exception, and it introduced ONE optional field:

```yaml
cost: 40
costModel:
  kind: potential   # m·g·Δh, added to the floor
```

A **closed** union, validated on catalogue warm exactly as `effects` is —
a row naming a model nobody implements is DROPPED rather than silently
priced flat. A second model is a design conversation, not a list edit
(the `SupplyState` rule, one level over).

`MagicLogic.costOf(traveller, spell, target)` is the one place cost is
computed, and both doors go through it:

- **the cast door** — `resolveCastImpl`'s inline expression became this
  call; `prepareCastImpl` now returns `costTau` so a front door can
  preview the full price without spending anything.
- **the item door** — ⚠ and this is where it mattered. Both item sites
  read `spell.cost` FLAT. Under a cost model the authored number is only
  the **floor**, so a flat read would have **undercharged every use** of
  a model-bearing item, by the whole physics term.

⭐ **The flat arm got simpler, not more complex.** It is byte-for-byte
the expression that was inline, and a spell with no `costModel` cannot
reach the second line — which is why the assertion that protects this
whole seam is a pin: `dispel` still costs exactly 20.

### `relocate` — the twelfth `Effect`

The closed union grew one member, backed by `Mobile.teleport`. ⭐ Its
executor lands on **`ctx.actor`** and has **no flag to say otherwise**,
which is how "a caster cannot teleport a third party" is enforced
STRUCTURALLY rather than by a check somebody can forget. An item in the
middle changes nothing: `EffectContext` separates `actor` (the wielder,
the reader) from `origin` (the wand), so a wand of teleport moves its
user rather than cloaking itself.

`to` is an item's **fixed survey** — a wand holds one place, the network
holds many, shared. Absent ⇒ the trigger's own target, which is what a
cast supplies. ⭐ That is why *a wand attuned to a mountaintop holds
fewer charges than one attuned to a valley*, and nobody tuned it.

See [fasttravel.md](./fasttravel.md) and
[magic-items.md](./magic-items.md).

## The governing invariant

> An `Effect` primitive exists **iff** a gated Api already does that
> work.

Enforced **structurally**: `Effect` (`lib/magic/Effect.ts`) is a closed
union; `MagicEffects.validate` throws on any unknown kind, and the
`SpellCatalogue` drops a spell whose effects fail validation at warm
time — "gain 5 levels" is unrepresentable, not merely forbidden. The
`script` kind is the exotic-5% trapdoor, **code-trust (`isWizard`)
gated** at execution.

## The pieces

| Piece | Home | What it is |
|---|---|---|
| Grid vocabulary | `lib/magic/Grid.ts` | 5 verbs × 13 active nouns (lightning + storm graduated from frontier when their substrates shipped; time + spirit remain) + `MagicProvenance` |
| Effect union | `lib/magic/Effect.ts` | the closed catalogue: `inject-channel` / `afflict` / `relieve` / `adjust-reserve` / `move` / `conjure` / `sense` / `cloak` / `emit-field` / `script` |
| Resist shapes | `lib/magic/Resist.ts` | axis vocabulary + the pure fold (`mitigators subtract`, immunity = a mitigator at 1) + `stageFor` (the substrate gate) |
| Faculty | `lib/magic/Faculty.ts` + `lib/magic/Caster.ts` | the anatomical casting faculty (below) |
| Suppression | `lib/magic/Suppression.ts` | the anti-magic field value + match + the **sync** outward containment walk (`fieldAt`) |
| Spell | `platform/idea/magic/Spell.ts` (data-Idea) + `platform/idea/SpellCatalogue.ts` | authored roster leaves (the arcane library's, under `/stuff/idea/magic/Spell/<spellId>`), boot-warmed **by class** (`Template.findByClass(Spell.CLASS_PATH)` — a pack ships spells under its own root with no kernel edit), **no Mongo collection** |
| The pair | `api/magic.ts` + `platform/idea/api/MagicLogic.ts` | the gated cast pipeline + all effect executors |
| Cast activity | `lib/magic/CastActivity.ts` | `hands`+`voice`, interruptible, **everything at completion** |
| Verbs | `cmd/magic/{cast,spells}.yaml` | the `magic` command category |
| Loci | the arcane library's `src/thing/GlowlightMote.ts` / `SparkLocus.ts` | glowlight's held mote / spark's transient energized locus — named by the spell row's required `locus:`; the executor clones what it is told (capability packs, D3) |
| Demonstrator | `world-seed/content/world/practicum*` | casting yard / conductive gallery / warded cell |

## The cast pipeline

`caster.prepareCast` (gates, spends nothing) → `CastActivity`
(hands+voice for the strain-slowed cast time) → at **completion**
`caster.resolveCast`: re-validate → spend → execute effects → stamp
provenance → credit the Transcript. An aborted cast spends nothing and
fires nothing (`magic.abortCostFraction` reserved at 0). **This is the
active gate**: interruption IS the shipped combat/engagement model —
there is no parallel resolution anywhere. The passive gate is the
resist seam.

Gates, in order: faculty active (`MixinApi.isCaster` — composed on
every `Character`, **activated** by `Species.innateMixins` or an
augment, the `isMaker` shape) → spell known → targeting shape →
casting-hand impairment (`isSlotImpairedByTrauma('hands')`) → the
**band gate on BOTH grid axes** (`requiredBand` vs
`competenceBandFor` on `magic-<verb>` + `magic-<noun>` —
competence IS access; the magic Disciplines — the 18 `magic-*` rows,
which ship in the **arcana** pack because its `Grid` derives their keys
(the discipline-ownership rule; the `DisciplineCatalogue` warms by class,
not by root) — carry **no `conferrals`**,
the `cast`/`spells` verbs ride the `refreshCastingAffordance` dynamic
self-push at `Avatar.enter` — the `refreshConferrals` mirror; the
`self` affordance bucket collects **class statics only**, so a gated
mixin cannot afford selectively via static or instance contributions)
→ suppression (deep tier).

Every cast credits **both axes** as two subchecks of one
`ActSignature`; potency scales by how far the caster's *limiting* axis
sits above the spell's floor (`magic.potency.competenceFactor`).

## The anatomical faculty

`CasterMixin` (`lib/magic/Caster.ts`, composed on `Character`, gated) +
`Species.facultyProfile` (`{depth, serenity, composure}` bands, the
`vitalProfile` field precedent):

- **mana** — a `Reserve` instance (`pt`, theme `'arcane'`), **absolute
  avail/max**: capacity derives from **depth**
  (`magic.depthCapacity.<band>`), costs are absolute points, % is
  derived where a fraction is wanted. Never a forked mechanism, never a
  stored CON-style scalar. **Read it via `getMana()` /
  `getManaFraction()`** — the contract surface, bundling the recovery
  reconcile a raw keyed `getReserve('mana')` skips (the
  `Combustible.getFuelRemaining` pattern; `MANA_RESERVE_KEY` is
  magic-internal plumbing — see the reserve landscape table in
  `lib/reserve.ts`).
- **recovery** — reconcile-on-read at the **serenity**-banded rate
  (`magic.recoveryPerMinBase` × `magic.serenityFactor.<band>`), scaled
  by the same rest inputs metabolism uses (posture base ×
  host `restQuality`), with the metabolism stamp guards (first-touch /
  linkdead freeze / far-past). Depth × serenity are orthogonal — a big
  tank refills no faster.
- **overchannel** — completing a cast past empty floors the pool and
  afflicts `overchannel-strain` (stage from the deficit), which slows
  casting until recovery clears it by hysteresis
  (`magic.overchannel.clearThreshold`).
- **composure** — the mental-axis resist substrate, read **live**:
  `composureBase(band) × (floor + (1−floor) × manaFraction)` — a
  drained mage resists fear worse. A species without the faculty reads
  the neutral default band (composure exists without casting).

Player surfaces (`spells`, refusals) speak **bands and prose, never
numbers**.

## The resist seam (N-axis)

The materials-response shape generalized: **mitigators subtract**
(folded outside-in; a mitigator returning 1 = immunity — the limit of
graded resist, not a special case), **the substrate gates** (sets stage
thresholds + picks outcome, scaled by live target state). The
load-bearing build finding: the **channel axis was
already fully packaged** behind `ConditionApi.inflict` (covering-stack
fold + tissue gate + banding), and **toxin** is the shipped metabolism
banding — so magic *delegates* those wholesale and implements exactly
one new resolver:

| axis | mitigators | substrate | outcome |
|---|---|---|---|
| channel | armor covering stack | tissue | Trauma — **delegated to `ConditionApi.inflict`** |
| mental | *(none v1 — wards are later content; the fold shape ships)* | **live Composure** scaling the condition seed's authored `mentalBands` | staged condition (dread) |
| toxin | — | metabolism burden banding | *(shipped; recognized, no v1 spell)* |
| none | — | — | lands in full |

Intensity units are per-axis (channel = real energy; mental = an
authored potency scalar) — no fake unified unit. The mental resolver
lives on `MagicLogic`; `Condition` grew the `mentalBands` authored-
cutoffs field (the `toxinBehavior.bands` precedent).

## Impulse vs modifier; provenance; suppression

Every effect is one of two **families**, derived from its kind:

- **impulse** — fired-and-released: real now, can't un-happen. A
  firebolt's burn, a landed dread (which then *decays* on
  `magic.dread.decayPerSec` via the magic reconcile arm).
- **modifier** — sustained, magically-bound: installs a
  **`SustainedEffect`** — the fourth `ActiveCondition` kind (the
  `SustainedShock` precedent) — realized **by pull** in
  `VitalsMixin.reconcileConditions`: glowlight = a bound conjured
  `GlowlightMote` (the row's `locus:`) whose flux the arm drives; veil = an imposed
  `Disguisable` disguise. Expiry / dispel → `releaseSustained`
  (un-realize, destruct the bound emitter, drop the record).

**Provenance**: everything magic produces is stamped
`MagicProvenance = {verb, noun, spellId, caster}` — on `Trauma`,
`AfflictionRecord` (both optional `magicOrigin`) and `SustainedEffect`
(required). Read by:

- **dispel** (`relieve`, tag-keyed ONLY — structurally unable to touch
  a mundane condition; traumas refuse too, an impulse can't un-happen)
  and **detect** (`sense` — arcane sight reads the tags);
- **the anti-magic field**: `Location.suppressesMagic`
  (`{all:true}` or a `verbs`/`nouns` grid filter), resolved by the
  **sync** outward containment walk (`Suppressions.fieldAt`, the biome
  chain shape) + an **async zone tier**
  (`Zone.lookupField('suppressesMagic')`) folded in at cast time
  (`suppressionAtDeep`). Casting inside is vetoed in `prepareCast`
  with legible prose; a **modifier** goes dormant on its next
  reconcile read (flux 0 / veil lifted — dormant, not released; it
  re-realizes outside). **The suppressible line IS the
  impulse/modifier line** — a mundane (or impulse-lit) fire keeps
  burning; there is no code path by which suppression touches an
  impulse.

## The ranged-integration seam + accountability

All **hostile channel-delivery** routes through ONE internal leg —
`MagicLogic.deliverAt` — the **ranged-integration seam, now adopted.**
The same-scene check STAYS (cross-room fire is out of scope, which is
what makes leaving through an exit a real escape); on top of it the leg
now asks a **band** question via `CombatApi.bandBetween`. The envelope
defaults to the `magic.spellEnvelope` dial, seeded `far` — i.e. anywhere
in the scene — so **every shipped spell behaves exactly as it did**, and
the gate bites only for a carrier that declares a tighter reach (a
thrown contact payload reaches `close`).

`DischargeOptions.origin` is the sibling of `source`, and load-bearing
for carriers: reachability measures from the ORIGIN, and a `Material`
singleton has no place of its own, so a thrown flask's payload would
otherwise issue from the *thrower* and refuse across the gap it just
crossed. See [ranged.md](./ranged.md).

After a landed damaging delivery on a non-consenting **sentient**
victim outside a shared combat session, `deliverAt` appends an
**accountability `harm` row** (`AccountabilityApi.record`, the
trap-spring producer precedent) — magic is inside the consent/blame
economy from day one. Inside a shared combat session the combat ledger
owns the encounter's rows (no double-booking).

## The v1 roster (authored data, one cell per primitive)

firebolt (create·fire, impulse heat — body burn / object
joules+autoignite) · spark (create·lightning — a transient
`SparkLocus` (the row's `locus:`) + the real conduction walk, faction-blind,
caster-in-the-graph) · shove (control·body — the posture surface) ·
dread (destroy·mind — the mental axis vs live Composure) · glowlight
(create·light — the modifier proof) · conjure-water (create·water —
real litres from a transient unbounded source into a vessel / the
floor pool) · veil (create·sense — imposed semblance, modifier) ·
dispel (destroy·arcana — tag-keyed relieve) · arcane-sight
(perceive·arcana — the detect read). Transform has no spell (no
backing Api — polymorph's own build); storm has a Discipline leaf but
no spell (no gated weather-write Api yet — the invariant holds it
back).

## The demonstrator — the Practicum

`world-seed/content/world/practicum*`: a teleport-reachable `CartesianZone` (the
hearthworks precedent, no inbound exit) — the **casting yard** (a
combustible straw dummy + a stone basin), the **conductive gallery**
(a brine-pooled floor: MIND WHERE YOU STAND), and the **warded cell**
(`suppressesMagic: {all: true}` + a mundane brazier that keeps burning
— the lesson in one room).

## Dials

20 `magic.*` AppSettings: `castSecondsDefault` / `costDefault` /
`abortCostFraction` / `depthCapacity.{low,mid,high}` /
`recoveryPerMinBase` / `serenityFactor.{low,mid,high}` /
`composureBase.{low,mid,high}` / `composure.floorFactor` /
`potency.competenceFactor` / `overchannel.severityPerDeficit` /
`overchannel.clearThreshold` / `dread.decayPerSec` /
`glowlight.lumens` / `conjure.waterLitres`. Shape stays in code;
magnitudes not yet worth a dial ride module `MAGIC_DEFAULTS` literals
(the `HARM_DEFAULTS` precedent).

## Deliberate boundaries / deferred

- ~~**Ranged delivery**~~ — SHIPPED. `deliverAt` carries a band
  envelope; see [ranged.md](./ranged.md).
- **Wards** as mental-axis mitigators — the fold ships N-axis; wards
  are later Arcana content.
- **The inquiry substrate** (discovery / predict / publish) — its own
  build; laws stay emergent here because effects ride honest functions.
- **Magic items** (Consumable / BUC / wands / scrolls) — the envelope
  is trigger-agnostic by design; build 11 consumes it.
- **Transform / polymorph**; **multi-cell composition** (steered
  firestorm); **Spirit / Time** frontier nouns; NPC caster brains;
  player-authored spells; the magical-material property layer
  (Part III resonance).
- The plan's separate `requiresUnsuppressed` validator was folded into
  `prepareCast` (one veto seam, same timing, better prose) — a
  simplification, not a scope cut.

> ⚠ **Deliberately untagged.** A spell's cost and duration look exactly like the numbers that got collapsed elsewhere, and magic here is a **published science** — one postulate, the laws, and *the price list* (see [arcane-science.md](../arcane-science.md)). A caster reasoning about the cost before paying it is the premise. A particular spell whose effect is a plot spoiler belongs behind a `<spoiler>` in authored prose, because a field level applies to every spell at once. The sweep that put `spoiler: 1` on
> `Material`'s and `Biome`'s measurements stopped here on purpose —
> *collapse what the WORLD measures, never what the PLAYER operates.*
> See [wiki.md](./wiki.md) § the reveal model.
