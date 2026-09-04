# Traits — the personality layer

The **trait substrate**: CK3-style opposed-pair dispositions modeled as
*competence for dispositions* — a character's personality is a **derived
aggregate over a ledger of disposition-valenced acts**, never an assigned
stat. Traits are the personality **input the NPC brains read** and the
**innate baseline for `regard`**. This is the trait analogue of the
[advancement](./advancement.md) measurement substrate, applied to
*character* instead of *skill*; the two share one authored `ActSignature`.

Home: `lib/trait/` (value-objects + the ledger Document), `api/trait.ts`
the owner face on `DispositionedMixin` (`lib/trait/Dispositioned.ts` — the Api OO sweep retired `TraitApi`/`TraitLogic`).

Scope of this build: the substrate + two read-only consumers (behavior +
regard baseline). **Stress / composure (job 3)** — the equanimity Reserve,
the break condition, the cope-drinking→tolerance spiral — is a deliberate
follow-on (the `traits-stress` build); see [reserve.md](./reserve.md).

## The model — derive-don't-track

A trait is **never stored**. The durable state is the append-only
`DispositionEntry` ledger; a character's position on each axis is **derived
on read** over it, exactly as `Competence` derives over the Transcript.
Re-tuning the estimator re-derives who someone is without rewriting a row.
There is **no stored trait field** — `DispositionedMixin` owns no fields; it
reads the owner-keyed ledger, mirroring how competence is read from the
owner-keyed Transcript.

### The roster — `lib/trait/Disposition.ts`

17 opposed-pair axes: CK3's *personality* core (13 direct keepers), 3
reframed for this world (`boldness` ← Brave/Craven; `fairness` ←
Just/Arbitrary; `worldview` ← Cynical/Zealous, recast idealistic/cynical
with no faith hook), and 1 native addition (`curiosity` ← Curious/Incurious,
central to a learning game). Everything CK3 handled via its *other* trait
categories is left to Saxonberg's own systems (education → advancement,
congenital → race, health → vitals). Each axis has a durable `key` and two
pole labels; `DISPOSITION_KEYS` is the validation array. **Polarity:** a
positive valence drives toward the `positive` pole.

### The ledger — `lib/trait/DispositionEntry.ts`

A plain `Document` (one row per disposition sub-check) in the
`disposition_events` collection, indexed on `owner`. The **sibling** of
advancement's `TranscriptEntry` (the renown↔regard / transcript↔chronicle
"sibling, not child" pattern): the `deed`/`claim` provenance *concept* is
reused, but the schema is `{disposition, valence}` (an opposed-pair axis +
a signed magnitude). `owner` is keyed on `templatePath`; `disposition` is
the durable axis `key` (not its templatePath), so a roster re-path leaves
evidence valid. Two ledgers, one authored signature — "instrument once."

### The estimator — `lib/trait/TraitPosition.ts` + `TraitBand.ts`

A pure static-method value-object (no state, no instances) — simpler than
advancement's BKT. Per axis:

- **position** = the signed Σ of `valence · decay(now − when)`, clamped to
  `[-100, +100]` (the regard range). Sign picks the pole; magnitude is how
  pronounced.
- **mass** = Σ of `|valence| · decay(...)` — the accumulated evidence
  weight, which drives the **`TraitBand`** lifecycle: `unformed` →
  `defined` → `entrenched`. A strongly Generous and a strongly Greedy
  character are both *entrenched* on `generosity`, at opposite poles.

Decay is **game-time** (the renown clock, not participation's real time):
character forms over an in-world lifetime. Each axis is independent this
build; cross-axis propagation is deferred.

**Entrenchment resists drift, emergently** (the resolved design decision):
because position is the *raw clamped sum*, an entrenched axis whose
consistent evidence has pinned it at the rail barely moves under more
same-direction evidence, while an unformed axis moves the full amount. The
clamp is the inertia — there is no explicit damping term. *Consequence /
limitation:* drift in the **opposite** direction (a reversal) is not
specially resisted; a normalized/bounded-by-construction position with
explicit inertia is the deferred richer model.

The dials are AppSettings-driven (`traits.decayHalfLifeDays`,
`traits.definedThreshold`, `traits.entrenchedThreshold`,
`traits.pronouncedThreshold`, `traits.compatibilityScale`), read in
`TraitLogic` and passed into the pure value-objects; fixed fallbacks serve
the unwarmed (unit-test / pre-boot) path.

## The owner face — `DispositionedMixin`

The Api OO sweep retired `TraitApi`/`TraitLogic`: the family lives ON
`DispositionedMixin` (composed into `Character` beside `PersonaMixin`),
P4-named per ledger — `imprintSignature` / `imprintDeed` /
`dispositionEntries` / `traitPositions` / `traitPosition` /
`pronouncedTraits` / `compatibilityWith` / `regardBaselineToward` /
`seedTraitClaims`. Writers are `SelfOnly`-gated and sealed (`@Final
@Unshadowable`); reads are ungated; callers narrow with
`MixinApi.isDispositioned`. The estimator + mint path formerly hosted
in the hot-reloadable `TraitLogic` singleton at `/platform/idea/api/trait`, gated
`FromModule("/api/trait#DispositionedMixin")`. Internals are module-private free
functions (no intra-singleton `this.x()` to trip the gate). Surface:

- `recordSignature(owner, signature, opts)` / `recordDeed(owner, subcheck,
  opts)` — append: fan **only** the `dispositionValence` channel of a
  shared `ActSignature` into rows (the `discipline` channel is
  advancement's). No-op without a durable owner key, a connection, or any
  disposition-valence on the act.
- `entriesFor(owner, disposition?)` — the owner-scoped reader.
- `positionsFor(owner)` / `positionFor(owner, axis)` / `pronouncedFor(owner)`
  — derive-on-read; `pronouncedFor` returns the *defining* axes the
  self-view and brains consume (most pronounced first).
- `compatibility(a, b)` — the trait-compatibility scalar (below).
- `regardBaseline(viewer, subject)` — the regard baseline (below).
- `seedClaims(owner, seeds)` — seed `claim`-kind evidence (below).

**Layering:** the trait layer depends on advancement (the `ActSignature`
*type* only — no runtime edge) and on belief's `BeliefStoreMixin (the regard face)` (one-way).
Neither advancement nor belief gains any dependency on the trait layer.
**Gated-API actor-from-context:** `owner` is the *subject of the act*
(established by the recording call site), never a spoofable actor param.

## Compatibility → the regard baseline (Job 2)

`compatibility(a, b)` is the scaled dot product of the two characters'
shared-axis positions: same pole on an axis contributes positively, opposed
poles negatively, a neutral side ~0; divided by `traits.compatibilityScale`
and clamped to the regard range. It is symmetric and derive-on-read.

`regardBaseline(viewer, subject)` is the trait-layer baseline: a stored
interaction-driven regard row **governs** once it exists (even at value 0 —
presence of a row means interaction has spoken); absent one, the innate
compatibility is the starting regard ("sets the *starting* regard…
interaction moves it from there"). It is **derive-on-read — nothing is
written to belief**, and belief never reads the trait layer. Consumers
(brains, future dialogue warmth) call `viewer.regardBaselineToward(subject)`.

## Behavior (Job 1) — the demonstrator brain

`lib/behavior/converses.ts` is the demonstrator: a trait-aware chatter
brain that reads `host.traitPosition("sociability")` and modulates
its speech — a Gregarious host holds court (warm `chatty` pool); a Shy host
mostly stays quiet, speaking tersely on the rare turn. This is the first
brain to read the trait layer, establishing the allowed **behavior → trait**
read edge (brains may read traits; the trait layer never reads behavior).
Wired onto the cast: Remy (gregarious, loud) contrasted with Mara / Sloane
(reserved, near-silent). Broad trait-awareness across the brain set is left
to the dialogue / follow-on builds (light touch by design).

## Cast personality at spawn — seeded `claim` evidence

Authored NPCs need their defining character *immediately*, but
derive-don't-track means a fresh ledger is near-neutral. The resolution
(mirroring char-gen's chronicle/transcript claim-seeding): seed disposition
**evidence**, not a stat. `BehavedMixin` carries a declarative
`dispositions: ClaimSeed[]` field; at `postRegister` it seeds those as
`claim`-kind rows via the host's own `seedTraitClaims` — **once** (idempotent across
re-clone / CMS go-live: it skips if any `claim` row already exists). So
Mara derives reserved & temperate, Remy gregarious, etc. — personality that
came from a seeded history, not a slider. (This is the deliberate
behavior → trait edge the demonstrator brain also establishes.)

## The self-view — `traits`

`cmd/charactergen/traits.yaml` + `platform/idea/cmd/charactergen/TraitsController.ts`
— a zero-arg, self-only, read-only verb (the `chronicle` / `competence`
shape), Persona-afforded via `Persona.commandContributions.self`. It renders
each pronounced axis as **pole label + band** ("Gregarious — entrenched"),
never the raw signed magnitude — the same honesty firewall competence keeps.

## Layering diagram

```
behavior (converses brain, BehavedMixin seeding)
   │  reads / seeds
   ▼
trait  (DispositionedMixin / lib/trait)
   │  reads ActSignature type            │  reads (one-way)
   ▼                                      ▼
advancement (ActSignature)            belief (BeliefStoreMixin (the regard face))
```

## Deferred

Stress / composure + the break condition + cope-drinking→tolerance (the
`traits-stress` follow-on); cross-axis evidence propagation; a richer
compatibility kernel (CK3-style cross-axis attraction); explicit
drift-inertia (the normalized-position variant); full-surface
disposition-valence authoring (only a starter set rides authored
`ActSignature`s today); npc-dialogue voice-from-traits (consumes the
readable trait-position).


## The standing witness

After each append is persisted, the ledger calls
**`MqlSubscriptionApi.notifyDurableSubject(subject)`** — a direct method
call on the one consumer that cares.

Every standing here derives on read, which works fine for a verb (you
ask, it computes) and not at all for a **live figure on a client**,
which has to learn its number changed without asking. This is that seam.

⚠ **It is deliberately NOT an `EventApi` broadcast.** The bus is for
genuinely global signals with unknown consumers; this has exactly one
known consumer, so it is a method call. An earlier cut of this build did
mint a bus event per ledger — six classes — and they were not merely
redundant, they were **wired to nothing**: the dependency index cannot
match a durable `templatePath` through a `ChangeSource`. See
[mql-subscription.md](./mql-subscription.md).

⚠ **After the write, never before.** A consumer that recomputes must not
read a ledger missing the row it was just told about.

## ⚠ Trait position and the live dashboard

Trait position is **not** a subscribable standing figure, and that is a
design constraint rather than an oversight — see
[mql-subscription.md](./mql-subscription.md) and
`docs/slates/builds/psychology-slate.md`. The vocation rests on
**self-other asymmetry**: you cannot read yourself, another person can,
which is *why the profession exists*.

⚠ **The `traits` and `score` verbs do self-report today.** That predates
the psychology slate's framing and contradicts its stated premise that
"the engine derives `TraitPosition` and shows nobody, so privacy is
free." Whether those verbs keep self-reporting is a **product decision
the psychology build has to make**; it is called out here so it is
found deliberately rather than rediscovered.
