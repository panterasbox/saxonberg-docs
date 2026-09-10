# Identity — who a character IS, and who answers for them

Every ledger in this game derives what it knows from an append-only
record, and every record has to attribute its rows to **something**. This
doc owns the question *what*: which key a row lands on, whether the thing
it lands on is a **person** or a **role**, and who else answers alongside
them.

⭐ Nothing here is a stat. A character's competence, personality,
standing and history are all still **derived on read** from evidence —
what this subsystem adds is the ability for an author to *state* a
history and have the world believe it exactly as if it had happened.

## The problem it solves

An authored character has done nothing, so the world reads them as having
done nothing. Dave, who runs a bar, was a novice bartender. The registrar
who is the city's entire civil service had no record of it. Every venue
this project has shipped was staffed by people who read as having never
done the job.

---

## ⭐⭐ The two rungs — `Cast` and `Extra`

A character is either **somebody** or **a role somebody fills**, and the
shipped prose had been saying which all along without being asked to. Of
39 written characters, 26 carry a proper name and the rest split on the
article: *a* sentry, *a* sellsword, *a* hewer on tutwork — against *the*
collier, *the* smelterman, *the* storekeeper.

> **The rung is: a proper `name:`, OR the definite article.** The build
> made the engine agree with the prose rather than introducing a new
> signal.

| | `Cast` | `Extra` |
|---|---|---|
| is | somebody — Odile, Dave, *the* collier | a role — *a* sentry, *a* sellsword, a wolf |
| singleton | ✅ `SingletonMixin`; a second live clone **throws** | ❌ two sentries are the point |
| dossier | ✅ | ⛔ a build error (`lint:identity`) |
| proper name | ✅ or the definite article | ⛔ a build error |
| harm attributes to | the person **and** the institution | the institution only |
| personality | ✅ | ✅ — a role is a mask, not a life, so it never changes |

⚠ **An `Extra` keeps its OWN identity.** It is not anonymous and it does
not project onto its institution: two dead sentries must not collapse
into one corpse. What an Extra lacks is a *person* to attribute to, which
is why the institutional attribution is a **second** attribution rather
than a replacement projection — an earlier draft of the design got this
wrong in both directions.

### It is a MIXIN, and that is forced

Identity and capability are **two axes**, and TypeScript has single
inheritance. Dave must be a `Crafter` *and* cast; base classes cannot
express that — it is a diamond.

⭐ The codebase had already answered it one line away:
`Crafter = MakerMixin(NPC)` and `Mercenary = PartyMemberMixin(NPC)` are
the capability axis *already* expressed as a mixin over the substrate and
given a name. So:

```ts
lib/npc/Cast.ts        CastMixin = SingletonMixin + the dossier fields
platform/agent/Extra   class Extra extends NPC {}                 // the plain rung
platform/agent/Cast    class Cast  extends CastMixin(NPC) {}      // the identity rung
platform/agent/Crafter class Crafter extends CastMixin(MakerMixin(NPC)) {}
```

`lib/npc/NPC` stays as the shared substrate. `platform/agent/NPC`
**retired** — after the split no row names it, and *a class nothing
instances does not belong in `platform/`*.

⚠⚠ **A correlation trap to keep refusing.** All seven `Crafter` rows
carry a proper name and the one `Mercenary` does not, so capability and
identity look perfectly correlated. **They are not.** That is a 39-row
accident of the same species as *every NPC row is instanced exactly
once*, and collapsing the axes on the strength of it would bake the
accident into the type system.

### Promotion is an authoring act

There is no runtime transition to build. Identity is a stamp and
`setTemplatePath` re-keys the registry index, so promoting an extra means
**authoring a `Cast` row**.

---

## The dossier — evidence, never values

One authored block per `Cast` row, holding every prior a derive-on-read
aggregate needs, expressed as **seeded evidence carrying the `claim`
marker**:

```yaml
archetype: proprietor          # ⭐ the stamp — see below
prologue:                      # → chronicle claims
  - "Did a little acting once, drifted through a lot of odd jobs, and
     finally settled down to open his own bar."
competence:                    # → transcript claims
  - { discipline: bartending, asserting: expert }
renown:                        # → seeded reception events + a fold
  - { asserting: familiar }
dispositions:                  # (stays on BehavedMixin — both rungs have one)
  - { disposition: boldness, valence: 70 }
```

⚠ **No `participation`, ever.** That zero is what keeps an authored
character out of the Compact, and it wants no rule to hold — see below.

Four properties that matter more than the shape:

1. **Evidence, not values.** Each entry expands into the same rows a
   lived history would have written, marked `claim`.
2. **Idempotent, once** — applied at `postRegister`, skipped when a claim
   already exists, so a re-clone, a reboot or a CMS go-live cannot mint a
   second history.
3. **It states its intent**, which makes it checkable — `lint:dossiers`.
4. **It never overrides a lived history.** A character written as skilled
   who then plays badly reads as somebody whose reputation exceeds them,
   which is a true thing about people.

### ⭐⭐ The archetype stamp — one field now, unrecoverable later

`kind` separates *authored* from *earned*. It does **not** separate an
**archetype claim** from a **deviation claim**, and those are different
things: an author who writes "a bartender, but unusually blunt" is
stating a departure from a baseline, and
`deviation = current derived − archetype baseline` is uncomputable
without knowing which baseline.

⚠ Provenance separability is the one property that **cannot be
retrofitted** — unwinding a sum with no seams is not possible later at
any price. The stamp rides `ChronicleEntry`, `TranscriptEntry` and
`DispositionEntry` alike.

### ⭐⭐⭐ The Compact stays players-only by ARITHMETIC

An authored character may be **well known** — that is a fact about the
world and the dossier writes it. What they may never have is **political
weight**. And that needs no rule:

```
standing = max(0, renownOf) × participationOf
```

Renown is the **quality** half (measured reputation — seedable, as
events). Participation is the **quantity** half, earned by turning up,
and nobody turns up on an NPC's behalf. So it is zero, and so is the
product. Dave can be famous in the lounge and politically weightless, and
nothing had to be written to make that true — no gate to forget, no
special case to maintain.

⚠ **On the doctrine.** *"Measure, don't assign"* forbids writing the
**figure**. It does not forbid authoring the **events** the figure
derives from, which is the dossier's founding move everywhere else.
`RenownApi.seedTo` appends `reception` evidence until the *current* value
function derives the asserted band, and never touches a standing row.

⚠⚠ **And renown needs the FOLD.** `renownOf` reads a warmed map off the
**materialized** `renown` collection, not off `renown_events` — so
seeding the log moves nothing until a recompute, and a bare restart
re-warms from a collection the seeding never wrote. That trap cost a
whole test-drive once. Seeding schedules the fold, **debounced**: 33
characters seeding at boot produce one recompute.

---

## ⭐ Who answers for you — the institution

The harm ledger names two persons (`killer` and `victim`); the
institution is their two **parties**. It lives on
{@link Employed.institutionPath} ([employment.md](./employment.md)) and
resolves in the chain shape this codebase uses everywhere:

1. an authored `institution:` — explicit wins;
2. else **the employer** — the first still-active `Employment`'s
   `organizationPath`;
3. else `null`.

⚠⚠ **Resolved from the DECLARED affiliation, never the current
location.** A guard who walks into a tavern does not become the
tavern's. There is no containment read on this path at all, and that is
the one way to get it obviously wrong.

⚠ **It is not `directedBy`** — that is *episodic* (a captain's order
began this act) where this is *standing* (you are fielded by X, order or
no order). And `victimFor` is **not crime-gated** where `killerFor` is: a
lawful duel that kills a guard is no crime against the watch, and it is
still a guard the watch lost. See
[accountability.md](./accountability.md) for the full asymmetry.

### ⭐⭐ A sentient `Extra` that answers to nobody is a BUILD ERROR

The crime rule is the terms-free `!consented && sentient`. So if hurting
something is a crime, the victim must be *someone*: either `Cast`, or
institutionally answerable. A wolf is fine forever — `sentient: false`,
and nobody is to blame for a wolf.

The shipped watchpost sentry was exactly this case, and its own row
documented the behaviour at risk: *"a player who ambushes the sentry
under lethal terms gets the imposed-terms crime marker."* That marker
named nobody. It worked only because there happens to be exactly one
sentry, so the shared row *was* an individual — the accident this
subsystem exists to stop relying on.

> ⭐ **The fix is content, never a weakened rule.** The gate narrows the
> question from *"why did my crime vanish"* to **"who fields this
> picket?"** — which an author can answer, and the answer improves the
> world instead of silencing an error.

⚠ `lint:identity` credits an authored `institution:` or a **Business**
roster, never a plain `Organization` roster. The roster tick enumerates
`mixin.BusinessMixin` and nothing else, so an Organization's
`rosterSlots:` materializes no `Employment` record; crediting one would
make the gate pass while the runtime resolve returned `null` — a gate
that ships broken and silently passes.

---

## Reading it back

Both readings take an **optional subject**, resolved by the shared
`RecordControllerBase` rule — *a subject must be addressed by a word it
calls itself* ([chronicle.md](./chronicle.md)):

| verb | reads |
|---|---|
| `chronicle [<subject>]` | a life on the record — or a body of people's losses |
| `competence [<subject>]` | the bands somebody has reached |

⭐ **The asymmetric gate now has a door.** An authored character's
competence is a fact about the world any viewer may learn; a **player's**
is their own, and asking is refused. Both verbs were zero-arg and
self-only before, so the permission was open and unreachable.

⚠ An `Extra` asked either question says so plainly — *"Nothing is written
down about a watchful sentry"* — rather than returning an empty answer,
so a role reads as a **role** and not as an empty person.

---

## The three gates

| gate | refuses |
|---|---|
| `lint:dispositions` | an authored disposition axis that resolves to nothing (five authored valences were landing nowhere), and an out-of-band valence the estimator silently clamps |
| `lint:identity` | a proper name on an Extra · a definite article on an Extra (or an indefinite one on a nameless Cast) · a Cast row instantiated twice · **a sentient Extra answering to nobody** · a dossier on an Extra |
| `lint:dossiers` | a dossier with no `archetype:` · an unknown Discipline · a band outside its vocabulary · ⭐ **an assertion the estimator cannot derive** · and a census-then-ratchet on dossier-less Cast rows (at **0**) |

⚠ `lint:dossiers` deliberately does **not** fold renown: its derive is
not a pure function of its seeds (AppSettings' value function, the Emote
documents' valences, the world clock), so the build-time fold that makes
the competence check worth having is unavailable. Its vocabulary is
checked at build time; its arithmetic at seed time, where `seedTo` writes
nothing at all if it cannot reach the band.

## The cast

| Element | Lives in | Role |
|---|---|---|
| `CastMixin` | `lib/npc/Cast.ts` | The identity rung: `SingletonMixin` + the dossier fields + the `postRegister` seeder |
| `Cast` / `Extra` | `platform/agent/{Cast,Extra}.ts` | The two generic clone targets, so a row's `class:` names its rung out loud |
| `Employed.institutionPath` | `lib/employment/Employed.ts` | Who answers for you — the two-tier resolve |
| `AccountabilityEvent.partyIdOf` / `partyForOf` | `lib/accountability/AccountabilityEvent.ts` | The one durable-id read, and the party read |
| `Competence.seedRunFor` | `lib/advancement/Competence.ts` | The inverse of the estimator: what evidence warrants a band |
| `RenownApi.seedTo` | `api/renown.ts` | Seed reception evidence to an asserted band, then fold |
| `RecordControllerBase` | `platform/idea/cmd/charactergen/` | The shared subject resolution behind both readings |
| `check-{dispositions,identity,dossiers}.ts` | `packages/server/scripts/` | The three gates |

## Cross-references

- [chronicle.md](./chronicle.md) — the prologue channel, the `chronicle` verb, the subject-resolution rule
- [advancement.md](./advancement.md) — the competence channel and the seeding ladder
- [accountability.md](./accountability.md) — `killerFor` / `victimFor` and the loss-vs-blame asymmetry
- [trait.md](./trait.md) — the disposition channel
- [renown.md](./renown.md) — the renown channel and the fold
- [employment.md](./employment.md) — where `institutionPath` lives, and why
- [behavior.md](./behavior.md) — `dispositions:` stays on `BehavedMixin`; both rungs have brains
- [mortality.md](./mortality.md) — a corpse's own identity
- lint-family.md — the three gates in the derived roster

## The drive

⭐⭐ **Tests build state; they never use it.** The live drive
(`packages/server/scripts/drive-identity.ts`,
`pnpm --filter @saxonberg/server drive:identity`) walks the requirements'
twelve steps — 18 checkpoints — over the real wire — the `test-login` seam, the same WebSocket the client
opens, the same command strings a player types.

⚠⚠ **It wants a freshly reset world, and this one is not negotiable.** A
written history is laid down ONCE, at the character's birth, and the
seeder skips a host that already carries a claim. A world booted before
the dossiers existed keeps whatever it was born with, so every checkpoint
reads as a broken feature in a build where nothing is broken. There are
no migrations; the reset IS the mechanism.

The founding run passed 18/18 and found four defects the 10,040-test suite
could not — three of them in the build's own new code, every one of which
would have shipped:

1. `competence dave` answered **"Dave's Bar"** (the Business Idea);
2. `chronicle the watch` answered **"a watchful sentry"** (MQL matches a
   prefix). ⭐ One rule fixed both — *a subject must be addressed by a
   word it calls itself*;
3. the casualty list printed **filesystem paths** at a player — a durable
   id is a key, never prose;
4. `Mml.unorderedList` was handed bare bodies rather than `<li>` items,
   so three bands rendered as one run-together blob (pre-existing in
   `chronicle`, invisible while a prologue was one line).

⚠ And the drive's own step 5 was passing on the **wrong branch** — it
asked about a player standing in another locality and accepted *"Nobody
here goes by…"*, the not-present answer rather than the refusal, so it
would have gone on passing with the gate deleted.

## History

**2026-09 · `design/dossier`, MR !248.** The subsystem's founding build —
five threads that turned out to be one question asked at five sites:
*what key does this ledger attribute to?* It closed issue #42
(accountability keyed on `templatePath` while every other ledger keyed on
identity), unblocked #40 (every corpse shared one identity), and filed
#43 (a definite article breaks every non-greedy object arg — found by the
drive, pre-existing, not this build's).

⚠ The institution read shipped as its own `AffiliatedMixin` and was
folded into `EmployedMixin` in review: a mixin whose composers are
exactly one class is the mixin-on-the-wrong-host tell, and both shipped
tiers are authored-or-employment. Split it back out if a third tier over
ground title lands, or if a non-employable host ever needs fielding.
