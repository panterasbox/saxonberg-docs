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
reseed). Keyed on the victim's durable **identity**
(`Stuff.getIdentityPath()`), so blame survives a reclone.

⚠⚠ **It keyed on `getTemplatePath()` until the identity build**, alone
among the ledgers — trait, transcript, chronicle, access, reactions,
channels and subjects all attribute to identity. The two are
byte-identical for every ordinary object and diverge exactly where
identity is *minted* or *projected*, which is where this ledger was
wrong: a sandbox `WireBody` is stamped `/platform/agent/Avatar/<id>/wire`
while *projecting* the player's real identity, so an in-circle harm filed
under the **vessel** — invisible to `blameFor(realIdentity)`, and
unreachable by the one reader that cares.

⭐ There is now **one read and no fallback** —
{@link AccountabilityEvent.partyIdOf}. Five producers had five different
substitutions for the same concept (`?? ''`, `?? 'stuff:<id>'`,
`?? stuffId`, a bare skip, and one that already read identity first), and
the `?? ''` one pooled every unattributable harm in the world under a
single key that `blameFor('')` would have read back as one person's
history.

⚠ **The empty string now means exactly one thing on a row:**
`AccountabilityEvent.NOBODY` — an act with no responsible party. An
environmental death writes it for `initiator` / `opponent` / `killer`
deliberately, which is why those fields cannot simply be required. The
**append seam refuses** a terminal row whose `victim` is empty (loudly,
and without writing), so the shared bucket does not exist to be read.
Closing it at the seam rather than only at the producers is what stops the
next producer reintroducing it.

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

### ⭐⭐ Every attribution has a PERSON and a PARTY

The identity build added the second half. The row already named two
persons (`killer` and `victim`); `killerFor` and `victimFor` name their
two **parties** — the standing institution that fields each of them,
resolved at write time from {@link Employed.institutionPath} (an authored
`institution:`, else the employer, else nobody).

⚠⚠ **This is NOT `directedBy`, and folding them together would be a lie
the governance design is careful never to tell by accident:**

| | |
|---|---|
| `directedBy` | **episodic** — a captain's recorded directive began *this act* |
| `killerFor` / `victimFor` | **standing** — this actor is fielded by X, order or no order |

A guard acting for the watch was not *directed* by the watch on this
occasion. Conflating them would make every institutional act read as a
command.

**The single-attribution case collapses into the existing field**, which
is what makes this two fields rather than a parallel ledger:

| the subject is | `killer`/`victim` carries | `killerFor`/`victimFor` |
|---|---|---|
| **`Cast`** — a person who also belongs to something | the person | the institution |
| **sentient `Extra`** — a role | the role's own row | the institution, the only attribution it has |
| **non-sentient `Extra`** — a wolf | the row path (*"a wolf"* is the honest unit) | `NOBODY` |

⭐⭐ **And the two are gated differently. That asymmetry is the design.**
`killerFor` is **crime-gated**, exactly like `commandResponsible` —
naming somebody's employer over a lawful duel is noise. `victimFor` is
**never gated**: a lawful duel that kills a guard is no crime against the
watch, and it is still *a guard the watch lost*. The actor side is about
**blame**; the victim side is about **loss**. Gate it on crime and a body
of people could only ever count its murdered, never its fallen — the
wrong instrument.

⚠ An `Extra` keeps its **own** identity (two dead sentries must not
collapse into one corpse), so the institution is genuinely a *second*
attribution rather than a projection that overwrites the first — which is
what an earlier draft of the design got wrong. See
[chronicle.md](./chronicle.md) for the `Cast` / `Extra` rungs.

## Surface — `AccountabilityApi` / `AccountabilityLogic`

The gated pair (`api/accountability.ts` forwarding shell +
`platform/idea/api/AccountabilityLogic.ts` at `/platform/idea/api/accountability`, gated
`FromModule('/api/accountability#AccountabilityApi')`):

- `record(fields)` — **fire-and-forget** append (a write failure never
  blocks the producing beat — a fight, a trap spring). The game-time /
  wall-clock defaulting happens here.
- `blameFor(victimId)` — the derived `BlameVerdict | null`.
- `crimeFor(victimId)` — the boolean shortcut (`blame?.crime ?? false`).
- `eventsForSession(sessionId)` — a producer's whole chain, `realAt`-ordered.
- ⭐ `institutionRecordFor(partyId)` — **a body of people's record**: the
  terminal harms it counted as `losses` (`victimFor`, never crime-gated)
  and the ones it is `blamed` for (`killerFor`, crime-gated). Circle-marked
  rows are dropped on both sides. It exists because blame was derived and
  **nothing player-facing showed it**, so *"the watch counts its losses"*
  was a claim nobody in the game could check; `chronicle <a body of
  people>` is the reader ([chronicle.md](./chronicle.md)).

Actor / consent / sentience ride in the row fields (durable **identity**
paths), set by the **producer that knows them** — never inferred in the
ledger.

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
- ⭐ **The bad meal** (the food-safety build) — a single `harm` row
  appended at the ingest (`Metabolic.noteMealAccountability`), on the
  trap's producer shape verbatim, when a `BulkPayload` carries a `maker`
  and the maker is **somebody else**. That last clause is the whole point:
  eating your own risky food is a private gamble; putting it in front of a
  paying customer is a choice about another person, and without the row
  the two acts would be indistinguishable. See
  [spoilage.md](./spoilage.md) and [metabolism.md](./metabolism.md).

  ⚠ The maker rides the **payload**, not the vessel. A dish reaches a body
  as `(material, litres, payload)` and the eater never sees the bowl, so
  the vessel's own `CraftedMixin` stamp could not answer for it.

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

## The identity build (2026-09) — what a ledger attributes to

`design/dossier`, MR !248. Three changes, and the first two were the same
bug seen from different sides:

1. **Keying moved to identity** (issue #42), with one read and no
   fallback, and the empty-string sink closed at the append seam. See the
   top of this doc.
2. ⭐ **`ConditionApi.die` RETURNED before the ledger write on the circle
   path**, so an in-circle death wrote **no row at all** —
   [sandbox.md](./sandbox.md)'s PASS(mark) classification (*"Identity-real
   … what happened to YOU stays yours"*) was dropping the one thing it
   promises to keep. The append is hoisted above the circle branch.
   ⚠⚠ Together with (1) this is what makes `deriveBlame`'s circle filter
   **load-bearing**: it had never been exercised by a row it could match,
   because no in-circle row was ever keyed on a real identity.
3. **`killerFor` / `victimFor`** — the party half of every attribution,
   and `institutionRecordFor` to read it.

⚠ Fixed in passing, and worth knowing about: `CombatLogic.safeSideOf`
keyed a solo side on `getTemplatePath() ?? ''`, so **two unidentified
combatants read as ALLIES**. It keys on `stuffId` now.
