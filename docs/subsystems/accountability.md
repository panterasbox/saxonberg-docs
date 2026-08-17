# Accountability (harm-consent ledger)

The unified answer to one question, however the harm was delivered:
**who harmed whom, on what terms, and did the victim consent?** A single
append-only ledger of harm facts with **derive-on-read consumers** — of
which `crime`/`blame` is the *only one this build ships* (the chronicle /
belief / renown "dumb store, smart consumers" shape). Combat, ambush, and
traps all feed it, so "you hurt someone who didn't agree to be hurt" is
*one* derived crime, not three parallel models.

This substrate **generalizes combat's former blame ledger**
(`combat_attribution_events` + `CombatAttributionEvent.deriveBlame`) and
**combat migrated onto it byte-identically** — its crime outcomes are
unchanged; its blame is now one *consumer* of the shared ledger. That
migration (proven by a pinned regression) is the load-bearing, riskiest
piece the concealment/stealth vertical rests on.

## The ledger — `AccountabilityEvent`

`lib/accountability/AccountabilityEvent.ts`. A plain `Document` (the row
IS the fact — the `RenownEvent` / `AuthoringEvent` precedent, not Stuff),
one row per attribution act in the `accountability_events` collection
(renamed from `combat_attribution_events`; pre-release, a trivial dev-DB
reseed). Keyed on the victim's durable `templatePath`, so blame survives a
reclone.

Four row **kinds** (the closed `AccountabilityKind` vocabulary):

- `opened` — a fight began: initiator, opponent, terms, consent, sentience.
- `violated` — the crime marker: lethal terms **imposed** on a
  non-consenting sentient (a standalone signal a future bounty consumer
  reads).
- `death` — a combatant was killed: victim, killer, and the terms in force.
- `harm` — a non-combat harm landed (a sprung player-trap): actor, victim,
  consent (non-consented by default — a snare is never agreed to),
  sentience. This is the one field-level addition; the combat-specific
  `lethality`/`stopCondition` are **optional** (harmless defaults on a
  `harm` row).

## The derivation — `deriveBlame` (branch on `kind`)

Culpability is **derived on read**, never a stamped stat: `deriveBlame`
replays a victim's rows, takes the **earliest terminal row** (`death` or
`harm` — the `ProvenanceApi.authorOf` earliest-row rule), and computes
`crime` by branching on that row's `kind`:

- `death` → `lethality === 'lethal' && !consented && sentient` — the
  **unchanged** combat rule (this is why the migration is byte-identical).
- `harm` → `!consented && sentient` — harm to a non-consenting sentient
  (there is no "lethal terms" concept for a snare).

Re-legislating what counts as a crime re-scores history without rewriting a
single row.

The combat-formations build grew the row by three optional **formation
facts** (`''` defaults, additive — no migration): `formationPath` (the
killer's side's formation in force), `killerRole`, and `directedBy` (the
captain whose recorded `fight finish` directive began the killing act).
They are facts, never crime inputs — the crime expression above is
untouched — but `BlameVerdict` grew the derived **`commandResponsible`**:
a crime row carrying `directedBy` names the commander alongside the
striker, so **credit and blame diverge** on an unlawful directed kill
(under Master-Apprentice the apprentice performs the coup and holds the
*deed*; the ordering master holds the *command responsibility*). A
directed formation implies command responsibility; the guard/law/court
consumers derive what to do about it. See
[combat-formations.md](./combat-formations.md).

## Surface — `AccountabilityApi` / `AccountabilityLogic`

The gated pair (`api/accountability.ts` forwarding shell +
`obj/api/AccountabilityLogic.ts` at `/obj/api/accountability`, gated
`FromModule('/api/accountability#AccountabilityApi')`):

- `record(fields)` — **fire-and-forget** append (a write failure never
  blocks the producing beat — a fight, a trap spring). The game-time /
  wall-clock defaulting happens here.
- `blameFor(victimId)` — the derived `BlameVerdict | null`.
- `crimeFor(victimId)` — the boolean shortcut (`blame?.crime ?? false`).
- `eventsForSession(sessionId)` — a producer's whole chain, `realAt`-ordered.

Actor / consent / sentience ride in the row fields (durable
`templatePath`s), set by the **producer that knows them** — never inferred
in the ledger.

## Producers (not a single chokepoint)

The natural temptation is to fold accountability into `ConditionApi.inflict`
(the common harm chokepoint). Rejected: `inflict` has no consent context,
and folding it in would sweep non-attributable environmental / thermal /
metabolic harm into the crime ledger (the wrong blast radius). Instead the
producers are the harm sources **that know consent**:

- **Combat** — its three writers (`opened` / `violated` / `death`,
  session-lifecycle events a per-inflict seam can't express) call
  `AccountabilityApi.record`; `CombatApi.blameFor`/`attributionFor`
  delegate to the shared ledger. See [combat.md](./combat.md).
- **The trap** — a single `harm` row appended at spring, co-located with
  the `inflict` call (`HazardMixin.deliverHarm` → `noteHarmAccountability`),
  so for the trap the producer site *is* the harm chokepoint. Only a
  **player-placed** trap (`placedBy` set) produces a row; an authored /
  environmental hazard has no culpable placer and appends nothing. See
  [hazard.md](./hazard.md) and [stealth.md](./stealth.md).
- **The ambush kill needs no new producer** — it routes through combat's
  normal death path with `consented: false` (imposed terms), so crime
  derives through the same ledger.

## Reserved consumers (named, not built)

The mandate is the *record of harm*, not the narrow "culpability" a
crime-only Api would imply — so reputation impact (a crime dents
renown/regard), self-defense/grievance, bounties (the newbie-wilds Law↔Chaos
thread), and courts/adjudication (the jury-pool docket) all read the **same**
rows without a rewrite. Loose-now, tight-seams-reserved; this build ships
the ledger + the one `crime` read.

## Collection

- `accountability_events` — the append-only ledger (`AccountabilityEvent`,
  one row per attribution act; indexed on `victim` and `sessionId`).

## Death is now the biggest producer (2026-07-31)

[mortality.md](./mortality.md) closed the ledger's largest gap: only combat
used to write here, so **eight of the nine ways to die left no record that
anything had happened to anybody**. Every death now appends, through the
single `ConditionApi.die` transition.

This does **not** make `die` a chokepoint in the sense this doc rejects.
The rule holds because the ledger still never infers: consent, killer and
terms are supplied by the producer that knows them. Combat builds its row
(`buildDeathRow` — the same fields as the old `recordDeath`, now returned
rather than written, which is what keeps `deriveBlame` byte-identical);
environmental drivers supply nothing, and `die` writes a row that **omits
`lethality`**, so it is structurally incapable of deriving as a crime.
That is stronger than claiming the victim consented to freezing to death.

Attribution rides the dying record, so a bleed-out finishing after the
fight resolves still names who struck — by then the session is gone.

**`deriveBlame` ignores circle-marked rows.** `circleScope` is now a
declared persistent field (it was being stamped by the persistence layer
and then silently dropped, because `fromDocument` only reads declared
fields) and stripped from field-side documents, so an ordinary row is
byte-identical to what it was. A killing staged in a private circle is not
evidence about anyone.
