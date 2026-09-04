# Advancement — the measurement substrate

Advancement is how a character **grows**: the **Transcript**, an
append-only ledger of practice, and the **Competence** bands derived from
it per **Discipline** — the engine-level "physics" that turns situated
practice into a current estimate of what a character is good at, and
confers verbs when that estimate crosses a threshold. This
build ships the **measurement layer** — Catalog, Transcript, Competence —
plus the conferral seam that makes growth observable, and a self-contained
proof harness. It is governed by the same **derive-don't-track** discipline
as [renown](./renown.md) and [chronicle](./chronicle.md): the estimate is
an *output you observe*, never an *input you set*.

Scope is the physics, not the content. The test for "is this physics":
*does it know what a sword is?* If yes it's content (deferred); if no it's
what this build owns. Loadout, guilds, the stakes engine, graph-propagated
evidence, estimator tuning, and per-verb signature authoring are all
deferred — see [Deferred](#deferred-seams-left-open).

## Three pieces

- **Catalog** — the authored, typed field-of-study graph. Each node is a
  `Discipline` (a field of study/practice at any grain); the graph is the
  impersonal shared canon.
- **Transcript** — the per-character append-only evidence ledger. Raw
  learning-events, never a score.
- **Competence** — derived on read over (Discipline × Transcript), **never
  stored**. Surfaced strictly as **bands** — the honesty firewall.

## Catalog — `Discipline` in a `DisciplineCatalogue`

A `Discipline` (`lib/advancement/Discipline.ts`) is a **pure-data leaf
`Idea`**, mirroring `Topic`: authored as a template under
`/platform/idea/Discipline/<key>`, read by the catalogue directly from
`template.data`, **never cloned as live Stuff**. Fields: `key`, `channel`
(`skill | knowledge | conditioning` — the procedural / conceptual / bodily
split), the three typed edge lists, and `conferrals`.

**`key`, not templatePath, is the durable join.** Edges and Transcript rows
reference a Discipline by its `key` (`'mixology'`), distinct from its
templatePath — so re-pathing / re-parenting the Catalog (additive
evolution) leaves edges and recorded evidence valid. Precedent:
`Topic.topic` / `Emote.verb` are domain keys separate from the path.

**Reality-seeded — the `iscedf` anchor.** A Discipline carries an optional
`iscedf` field: the UNESCO **ISCED-F 2013** field-of-education code it
classifies under (e.g. `'1013'` Hotel, restaurants and catering for the
bartending cluster; `'1014'` Sports for darts; `'0416'` Wholesale and
retail sales for appraisal). This is the same reality-seeded pattern
materials (`formula`) and species (`binomial`) follow — the impersonal
canon references the standard the world's learning institutions actually
use. `''` when there is no field-of-education anchor (a `conditioning`
Discipline like alcohol-tolerance is bodily adaptation, not a field of
study). Authored and stored; **inert in v1** (designed in for the academy /
LMS bridge, which will map real coursework onto Disciplines by code).
Authoring the *full* ISCED-F parent spine — broad → narrow → detailed field
nodes the bar leaves `specializes` under — is deferred content; v1 anchors
the leaves to the standard via the code.

**Edges are fields on the node** (`requires` / `specializes` /
`synergizes`, each a list of Discipline `key`s) — mirroring `Entry.relation`,
no separate edge documents. They are **authored and stored**; their runtime
*consumption* (prerequisite gating, evidence propagation) is deferred. The
graph is real and queryable; the estimator just doesn't walk it yet.

`DisciplineCatalogue` (`obj/DisciplineCatalogue.ts`, `/platform/idea/DisciplineCatalogue`,
bootstrapped) is the `TopicCatalogue` recipe: `PostRegistrationMixin(Idea)`
warming a `Map<key, DisciplineDescriptor>` in `postRegister` from
`Template.findDescendants`. Read-only canon — no runtime mutation surface
this increment. Unlike `TopicCatalogue` there is no derived fallback: a
Discipline either exists in the canon or it does not.

## Transcript — a sibling store, not a chronicle realm

`TranscriptEntry` (`lib/advancement/TranscriptEntry.ts`) is a plain
`Document` in its own `transcripts` collection (owner-indexed, one row per
sub-check — the chronicle spine, the `ContactsMixin` whole-array
anti-precedent avoided). It is the **sibling** of the chronicle, not a
realm inside it: the `deed` / `claim` provenance *concept* is reused, but
the schema is the structured learning-event
`{owner, kind, when, discipline, difficulty, outcome, tags}` — not
chronicle's narrative `text` / `order` / `who`. The renown↔regard
"sibling, not child" relationship.

- **deed** — a world demonstration (you *did* the thing); minted by a
  consumer loop at the moment of resolution.
- **claim** — a study / LMS attestation (you *studied* it); the academy
  faucet. Defined for the deferred learning-platform bridge; no consumer
  mints claims this increment.

`owner` is the durable character `templatePath`; `discipline` is the
Discipline `key`. The Transcript is *what happened*; re-legislating the
estimator re-scores history without rewriting a row.

## The act-signature — the unit of credit (cross-lane seam)

`ActSignature` (`lib/advancement/ActSignature.ts`) is the authored
decomposition of one messy in-world act into per-Discipline **sub-checks**:
`{discipline, difficulty, outcome}` triples (an authored Q-matrix). The
decomposition isn't inferred — it's *how the action was built*; the engine
already runs the sub-checks to resolve the action, and the Transcript just
records each as its own row (the owner's `creditSignature` explodes a
signature into N rows sharing one timestamp).

**One signature, two outputs.** The same signature carries an optional
`dispositionValence` channel — because traits are "competence for
dispositions" (the identical derive-from-a-behavior-ledger architecture
applied to character). **Lane 3 populates and consumes only the
`discipline` channel**; `dispositionValence` is declared-but-unpopulated,
the seam lane 1's trait build grafts onto without reshaping the type. See
npc-behavior-slate § *Traits are
competence for dispositions*.

**Difficulty is a world-measurement, not a tag** — the route's length, the
lot's ambiguity, the live competition. You can't farm a hard attempt by
relabeling.

## Competence — derive-on-read, bands only

`Competence` (`lib/advancement/Competence.ts`) is a **pure, stateless
value-object**: `Competence.derive(evidence)` folds a Discipline's
Transcript rows (in `when` order) into `{theta, band}`. **Nothing is
stored** — unlike renown's materialized `RenownStanding`, there is no
aggregate collection and no recompute schedule. Drop nothing, replay the
evidence, get the identical estimate.

The estimator is a per-Discipline two-state **BKT** with two difficulty
couplings that buy the desirable-difficulty property for free:

1. **Difficulty-modulated observation** — harder checks carry higher `slip`
   and lower `guess`; the easy end has a *high* guess (almost anyone passes
   a trivial check). So a trivial success and a formidable failure are both
   *unsurprising* and barely move the estimate; near-edge evidence
   dominates. Grinding trivial checks is a leveling-mill the math forbids.
2. **Difficulty-gated learning (ZPD)** — the BKT transit (learn) rate is an
   inverted-U over difficulty (~0 for trivial, peaking at hard): *how much
   you learn* tracks practicing at the edge of your ability.

Each Discipline is estimated **independently** this increment (no edge
propagation). The BKT parameters are fixed constants; numeric calibration
is deferred.

**The honesty firewall is "no quantity without a referent."** The internal
`theta` has a referent (competence *in a Discipline*) and may be read by
future instruments, but the **only surface is the band**
(`untrained | novice | competent | proficient | expert`, in
`lib/advancement/CompetenceBand.ts`). The owner's `competenceBandFor` /
`bandsFor` return bands; the scalar never crosses the Api boundary. You
learn you've improved the way you would in life — the lock that beat you
last week clicks open — not by watching a meter.

## Conferral — the knowing→doing seam

Crossing a competence band **confers verbs** (the way membership and
augments do). A `Discipline` declares `conferrals: {band, verbs}[]`; at or
above `band`, the `verbs` (yaml-paths) become afforded.

Because Competence is derive-on-read, a band crossing has **no event of its
own** — the only band-mover is a Transcript append. So
every `creditSignature`/`creditDeed` re-invokes the owner's
`AdvancementMixin.refreshConferrals` after every append (narrowed via
`MixinApi.hasMixin(owner, Mixins.Advancement)`). The refresh reads
the owner's `conferredVerbs` (the pure band × Catalog-conferral
decision), resolves the yaml-paths to `CommandDefinition`s, and reconciles
the giver's affordance stack — `popCommandSource` then a conditional
`pushCommandSource`, mirroring the hosted-update delta. The affording
`source` is the `DisciplineCatalogue`, so `getAffordances` attributes the
verb to "your competence in the catalog." A read-time-dynamic
`getAffordances` was **rejected** — it's a hot sync path and a band read
touches Mongo.

`AdvancementMixin` (`lib/advancement/Advancement.ts`) is composed
**outermost** on `Character` (above `CommandGiverMixin`, whose surface it
pushes onto). It holds no per-character state (advancement is
derive-on-read), only the static self-view contribution and
`refreshConferrals`.

## The owner face on `AdvancementMixin`

The Api OO sweep retired `AdvancementApi`/`AdvancementLogic`: the family
lives ON `AdvancementMixin` (`lib/advancement/Advancement.ts`), P4-named
per ledger — `creditSignature` / `creditDeed` / `transcriptEntries` /
`competenceBandFor` / `competenceBands` / `conferredVerbs`, plus the two
sync fold-cache reads (`practisingCompetenceCached` /
`competenceDigestCached`, over `DerivedStandingCache`). The credit
mutators are UNGATED with the seal (the writer set is every acting
controller and engagement, kernel and packs alike — an honest ungated
seal beats a FromModule glob) and end with `refreshConferrals()`.
Callers narrow with `MixinApi.isAdvancing`.

### The retired split (historical)

The gated `Api` ↔ HMR logic-singleton split (the `ChronicleApi` /
`ChronicleLogic` boilerplate; connection-gated via `PersistApi`,
`lint:pm`-locked). Surface:

| method | role |
|---|---|
| `recordSignature(owner, signature, opts?)` | explode an act into per-Discipline rows; trigger the conferral refresh |
| `recordDeed(owner, subcheck, opts?)` | single-Discipline `deed` convenience |
| `entriesFor(owner, discipline?)` | owner-scoped (optionally per-Discipline) raw reader |
| `bandFor(owner, discipline)` | the derived band for one Discipline (bands only) |
| `bandsFor(owner)` | a band per Discipline with evidence (the self-view read) |
| `conferredVerbs(owner)` | the verb yaml-paths the current bands confer |
| `practisingCompetenceCached(owner)` | **sync** — the one Discipline being practised |
| `competenceDigestCached(owner)` | **sync** — the whole projection (the digest) |

### ⭐ The competence digest

`practisingCompetence` answers *what am I working on*;
**`competenceDigest`** answers *what do I know* — every Discipline with
evidence and its band. Both ship as subscribable fields on
**`AdvancementMixin`** — the mixin that owns the subsystem, not on
`Avatar` — so the client's self-view reads them live.

### ⭐ Competence is expressed uniformly for players and NPCs

`ownerKey` is `getIdentityPath()`, so an NPC has always been able to own
a Transcript, and `Character` composes `AdvancementMixin`. Putting the
descriptors on the mixin is what makes that reachable: Dave the Barkeep
has the same competence fields a player does.

The read gate is deliberately **asymmetric**:

| Host | Who may read |
|---|---|
| player-controlled (`getPlayerId() !== null`) | **self only** |
| anyone else (an NPC, a fixture) | **any viewer** |

A player's competence is their own. An NPC's is a fact about the world —
Dave being good behind a bar is a thing you can learn about Dave. A
self-only gate on both would make the field *defined* on every host and
*answerable* on none, because an NPC never subscribes on its own behalf.

⚠⚠ **This is uniform EXPRESSION, not uniform authoring.** Nothing writes
an NPC's Transcript except combat (`CombatLogic` records against
`actor` / `state.combatant`), and the only authoring path is `practice`
— a wizard, self-only dev harness with no target argument. So most NPCs
read as the floor in every Discipline, and **there is currently no way
to state that Dave is good at bartending.** That needs an authoring
path, and the fork is whether an authored competence is *declared* on
the NPC/species (a floor `bandOf` maxes against) or *seeded* as
synthetic Transcript rows — which differ on whether `bandOf` stays a
pure derivation over real evidence, a property this whole subsystem
leans on. Unresolved; the read side is already correct for when it
lands.

⚠ `undefined` from these descriptors is **ambiguous by design**: it
means both *the gate withheld this* and *the fold has not landed yet*.
A subscription treats them alike (omit the field), but a caller that
needs to observe the gate must prime the fold first.

They are the **sync** faces of an async ledger: `bandsFor` awaits
`transcripts`, and a `subscribableFields.read` cannot. Each rides a
`DerivedStandingCache` whose loader does the async fold and whose
invalidation is the ledger's own `notifyDurableSubject` poke —
`undefined` means "the fold has not landed yet", which is a legitimate
answer the descriptor passes straight through.

⚠ **Derive-on-read; there is no stored total.** The band is already a
derivation over an append-only ledger, so a cached *total* would be a
second source of truth for a number the ledger owns, and the two would
diverge the first time a conferral landed without going through it. The
cache above is a **fold cache**, not a stored figure: dropping it costs
a recomputation and loses nothing.

## The proof harness

Lane 2's real craft verbs don't exist on this branch, so the loop is made
exercisable standalone:

- **`practice <discipline> [difficulty] [outcome]`** (`cmd/author/`,
  wizard-gated via `requiresWizard` + AuthorMixin visibility) — the
  harness that fabricates one `deed`, standing in for real practice.
- **`competence`** (`cmd/charactergen/`, the `chronicle`-verb shape) — the
  zero-arg, self-only, **bands-only** self-view; afforded by
  `AdvancementMixin`'s static `commandContributions.self`.
- **`flourish`** (`cmd/social/`) — a self-contained placeholder verb the
  Mixology Discipline confers at `competent`. It is in **no** static
  contribution, so seeing it appear IS the demonstration that advancement
  opened a door.

Seed Catalog (`seeds/lib/advancement/Discipline/`): the Dave's-Bar
practiced leaves — `bartending` (school), `mixology` (`requires`
recipe-knowledge, `specializes` bartending, confers flourish at
`competent`), `recipe-knowledge`, `appraisal` (`synergizes` mixology),
`darts`, `alcohol-tolerance` (conditioning) — hung under a real **ISCED-F
spine** of structural field nodes: a Services branch
(`services` 10 → `personal-services` 101 → `hospitality-catering` 1013 +
`sports` 1014) and a Business branch (`business-admin-law` 04 →
`business-administration` 041 → `retail-sales` 0416). The result spans all
three channels, all three edge kinds, and a five-grain chain
(`mixology` → `bartending` → `hospitality-catering` → `personal-services`
→ `services`) — the reality-seeded taxonomy made visible. The structural
field nodes carry `channel: knowledge` (a field of study is a knowledge
domain; the skill / conditioning distinction is a leaf attribute) and the
`alcohol-tolerance` conditioning leaf carries no ISCED-F code (bodily
adaptation isn't a field of study). The slice is illustrative, **not
exhaustive** — the full ISCED-F is content authored over time.

Later builds seed their own leaves the same way (pure data, zero code): the
combat build's `melee-combat` / `blades` (the bar-fight build adds
**`unarmed`**, the fisticuffs sibling of `blades` — credited on an
innate-instrument exchange), and the concealment build's
**`awareness`** (`channel: skill`, `iscedf: "0288"`) — the perception /
attention Discipline that grades a viewer's effective perception (the
`capacity` term), the reach of an active `search` / `examine`, and `disarm`.
It carries **no `conferrals`** (`search` / `look` / `disarm` are universally
afforded; competence only *grades* them, it doesn't gate the menu). See
[concealment.md](./concealment.md).

The stealth & deployables build seeds `awareness`'s opposed sibling —
**`stealth`** (`channel: skill`, `iscedf: "1032"`, no `conferrals`) — the
hiding face of the same contest: it grades a hider's derived concealment
level (`PerceptionApi.hideLevelFor`) and a placed trap's concealment, where
`awareness` grades the watcher trying to see through them. One contest, two
competences. See [stealth.md](./stealth.md).

The magic build seeds the **grid** — 18 leaves (`magic-{create,destroy,
control,transform,perceive}` verbs × `magic-{fire,…,storm}` nouns, each
`channel: skill`, `iscedf: "0288"`, verbs `synergizes` every noun), the
biggest single Catalog contribution yet. Deliberately **no `conferrals`**:
the `cast`/`spells` *verbs* ride `CasterMixin.refreshCastingAffordance`
(the `refreshConferrals` mirror — a dynamic self-push at `Avatar.enter`,
since the self bucket collects class statics only), and per-spell access
is a **band gate at cast time on BOTH of a
cell's axes** (`requiredBand` vs `bandFor` — competence IS access), so a
spell's floor rises with its power without a conferral row per spell.
Every cast credits both axes as two subchecks of one `ActSignature`. See
[magic.md](./magic.md).

The combat-formations build seeds **`command`** (`channel: skill`,
`iscedf: "0413"`, `synergizes: [melee-combat]`, no `conferrals`) — reading
a fight and directing others through it. Its deeds mint only for
formation policy work performed *for someone else* (an interception
taken, an armed opening an ally cashed, a captain's call landing, the
coup directive, the formation shift), so a master/captain advances the
one discipline that cannot be ground solo — the teaching payoff. See
[combat-formations.md](./combat-formations.md).
## Disciplines the food-safety build added

- **`butchery`** — `specializes: cooking`. Two things answer to it and
  neither is a damage number: **how much** meat a carcass yields, and
  **how much of the animal's gut ends up on it**. Gut spillage is the
  dominant real contamination route and it is exactly what an unskilled
  hand does, so one band read decides both. ⚠ An expert still deposits a
  floor of it — the answer to that hazard has to be cooking and cold,
  never a good enough butcher. See [spoilage.md](./spoilage.md).

  ⭐ It is its own Discipline rather than a use of `cooking`, and that is
  the spin-out seam: when ranching brings volume and `trade-butchery`
  leaves `trade-cooking`, the move carries a verb and a controller — not a
  skill model.

## Deferred (seams left open)

The loadout (capacity-not-decay, savings warm-up); guilds (venue / mentors
/ credential / membership-as-affordance); the Reserve-shaped stakes engine
(transient-deficit + played-recovery); **graph-propagated evidence** (the
Bayesian-net over `requires` / `synergizes` — edges are stored, unread);
the disposition-valence channel (lane 1's traits); per-verb signature
authoring across the command system; the skill-signature review gate;
learned signatures; estimator family/tuning; the raw-θ "spoiler" view; the
learning-platform sensor bridge (the `claim` faucet's producer); and
session-restore of conferred verbs at login (the evidence persists; a
login-time `refreshConferrals` wires it). All consumers beyond conferral
(guild exams, instrument reads, reputation) are deferred — the substrate
ships designed-for-them, ships none.

## Cross-references

- Seeding slate: advancement-slate.md.
- Precedents: [chronicle.md](./chronicle.md) (the append-only ledger),
  [renown.md](./renown.md) (derive-don't-track; Competence diverges by not
  materializing), [augmentation.md](./augmentation.md) +
  [command-routing.md](./command-routing.md) (the conferral / affordance
  seam), [persistence.md](./persistence.md) (`PersistApi` / `lint:pm`).
- Cross-lane: npc-behavior-slate.md
  § *Traits are competence for dispositions* (the shared act-signature).


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
