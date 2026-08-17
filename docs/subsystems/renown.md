# Renown — the measured-standing substrate

Renown is the **measured aggregate of social standing** — a signed scalar
(esteem ↔ notoriety), per scope, computed from how others actually react
to and receive you. It is the "quality" half of the cooperative slate's
consumer-influence thesis (`engagement × renown`). The governing stance:
**measure, don't assign** — renown is an *output you observe*, never an
*input you set*. This subsystem ships the substrate (the signal log, the
per-scope aggregate, the read API, two signal generators); it wires **no
consumer** — governance influence, NPC behaviour, and disguise/notoriety
read `RenownApi.renownOf` later.

Successor to reactions: it consumes the purpose-built `ReactionFiredEvent`
that reactions shipped with no consumer, and adds a second input — passive
*reception*.

## Where renown sits — the reputation family

Two **axes** (esteem / orientation) over two **layers** (subjective /
aggregate), fed by two **substrates**:

| System | Axis · Layer | What it is | Fed by | Status |
|---|---|---|---|---|
| **Contacts** (`ContactsMixin`) | — · subjective | my private *filing* of others | me only (declared) | shipped |
| **Regard** (belief realm) | esteem · subjective | my signed *attitude* toward you | reaction events → belief store | shipped |
| **Renown** | esteem · aggregate | your *standing* per scope (signed) | the **renown event log** | this build |
| **Susceptibility** | esteem · authored | how easily *this NPC* is swayed | authored NPC knob | deferred |
| **Alignment** | orientation · aggregate | your *stance* (Law axis) | the **chronicle** (deeds) | deferred |

*Reputation* is the family name for the esteem axis (regard + renown +
notoriety + susceptibility), not a separate quantity.

Two **dumb stores, smart consumers** — the same pattern twice:

- **chronicle** = narrative deeds → identity readouts (alignment,
  achievements, traits). Curated, permanent.
- **renown event log** = ambient signal → standing readouts (renown,
  notoriety). Bulk, decayable.

The data-flow is **acyclic in the dangerous direction** — that acyclicity
*is* the anti-capture guarantee:

- **Contacts feed nothing** — pure subjective lens (unilateral
  self-declaration = zero objective signal); they *consume* notoriety for
  display, never produce standing.
- **Regard → renown → influence**, but renown only ever *multiplies a
  bounded* influence (conduct → weight, never → authority).
- **Renown reads only its own log** — never the belief store (sibling of
  regard, not parent/child; see below).

## The two-layer store

- **`RenownEvent`** (`lib/standing/RenownEvent.ts`, collection
  `renown_events`) — the append-only, **scope-tagged** signal log; the
  source of truth. One `Document` row per signal, storing the **raw,
  pre-valence** signal (no score) so re-legislating the value-function
  **re-scores history without rewriting a row**. Indexed on `subject` +
  `{subject, at}`. Fields: `subject` (who it's about), `source` (who
  emitted it — retained for the future eigenvector weighting; a
  self-reaction is `source === subject`, not stored), `kind`
  (`reaction` | `reception` | …), `signal` (raw kind-specific payload),
  `locality` + `groups` (the two scope axes), and **both clocks** — `at`
  (game-time seconds, drives decay) and `realAt` (epoch ms, analytics).
  `subject`/`source` are the durable `templatePath` (`/obj/Avatar/<playerId>`
  for an avatar), **not** the ephemeral `stuffId` — the influence build's
  Phase 0 durability re-key: the `stuffId` is re-minted on re-clone, so it
  stays a live-resolution handle (scope resolution) while the *stored* key is
  the templatePath. (Value objects now live in `lib/standing/` — the
  consolidated home; see [influence.md](./influence.md).)
- **`RenownStanding`** (`lib/standing/RenownStanding.ts`, collection
  `renown`) — the materialized per-`{subject, scope}` aggregate; a
  **rebuildable cache**, never authoritative (drop it, replay the log,
  identical standings). Reads hit an always-a-`Map` in-memory cache
  (cold = neutral `0`, never throws) warmed at boot (`warm()`, mirroring
  `AppSettings.warm`) and refreshed by each recompute. `scope` is the
  stored key — the sentinel `'*'` = Compact-wide (the scope governance
  reads), else a `Group` ref or locality prefix. Stamps `recomputedAt` +
  `recomputedRealAt`.

## `RenownApi` / `RenownLogic` — the gated seam

`RenownApi` (`api/renown.ts`) is the thin gated facade; the logic lives in
the HMR-able `RenownLogic` singleton (`obj/api/RenownLogic.ts`, gated
`FromModule('/api/renown#RenownApi')`, internal sub-logic in
module-private free functions to dodge the gate on `this.x()` self-calls).
Surface:

- `append(fields)` — raw, pre-valence append; `at`/`realAt` default to the
  two clocks. No-op without a live connection (`PersistApi.isConnected`).
- `eventsFor(subject, scope?)` — the raw scope-filtered log reader
  (substrate seam). Scope containment (`inScope`): `null` = every event;
  a `Group` ref matches the `groups` axis; a locality matches the exact or
  any nested `locality`. The one definition the reader and recompute share.
- `recompute()` — the batch aggregator (below).
- `renownOf(subject, scope?)` — the **sync cached read** consumers call;
  neutral `0` for a non-materialized scope.
- `measuredRenownOf(subject, scope?)` — ⭐ the **absence-preserving
  companion**: the same read without the `?? 0`, so an unmaterialized
  scope answers `undefined` and a standing genuinely measured at zero
  answers `0`. **`renownOf` for scoring, `measuredRenownOf` for
  display.** A caller about to do arithmetic is right to treat a missing
  scope as contributing nothing; a caller about to *show* the figure
  would be printing a confident zero for a standing nobody ever
  computed. It is additive rather than a change to `renownOf` because
  all five of that method's callers genuinely want a number, and pushing
  the `undefined` outward would reproduce the same collapse four times,
  further from the data. `Avatar`'s `renown` subscribable field is the
  first consumer — see
  [mql-subscription.md](./mql-subscription.md).
- `boot()` — installs both ingestion taps + self-registers the recompute
  schedule (idempotent). Each tap **receive-gates** its event
  (`EventApi.restrictSubscribe`): `ReactionFiredEvent` and
  `CommReceivedEvent` carry per-actor activity (who-reacted-to-whom,
  who-heard-what), so their subscribe side is locked to `RenownLogic` — an
  open-subscribe broadcast would be a snooping side-channel. Emit stays
  open. See [participation.md](./participation.md) for the apparatus.

## The two signal kinds

### `reaction` — an active, signed signal

`RenownLogic` taps `ReactionFiredEvent` (the renown seam reactions
shipped). Per fired reaction it appends a scope-tagged
`kind:'reaction'` event carrying the raw emote + tags. **`ReactionRegistry`
stays ephemeral** (5-min TTL, live UI chips) — one event, two consumers,
one ephemeral, one durable. Valence is **per-emote**, on the emote itself
(`Emote.valence`, signed; seeded in `emotes.yaml` — cheer/applaud/clap/
laugh/grin/agree `+1`, scowl/glare/frown/disagree/groan `−1`), read at
recompute via `SoulApi.all()`. Reactions score **linearly**.

### `reception` — passive being-heard

A small engagement reward for the speaker, measured on the **receiving
end**: `CommReceivedEvent { perceiverId, commandId }` fires from
`SensorMixin.onMessage` **after `filterMessage`** — so it is genuinely
perception-gated (a deaf / shadowed / disconnected listener never fires
it) and un-farmable by monologue. The fire is **gated to communication
topics** by the data-driven `communicative` flag on the `Topic`
(say/whisper/shout/emote/chat — **not** dm/narration/system), consulted
via `MessageApi.isCommunicative` (→ `TopicCatalogue`'s `communicative`
set), so non-comm frames never hit the bus. The reception tap recovers the
speaker + scope from the reactable-act registry (`ReactionApi.actInfo`),
skips non-comm frames (`null`) and self-receipt, and **dedups per
`(speaker, listener)`** within `renown.receptionWindowS` (reward reaching
*new* people, not repetition). At recompute, receptions are
**log-saturated** — `receptionValence × ln(1 + Σ decayed)` — so the first
public messages matter far more than the thousandth (the diminishing-
returns the sociology predicts: mere-exposure / weak-ties). It fires per
genuine receipt (N per utterance); a debounce on that hot path is a future
optimization.

## The value-function

The polity sets **declarative parameters in a fixed engine algorithm**
(the constitution-parameters shape), partitioned by **entrenchment tier**
— the tiering *is* the anti-capture mechanism:

- **Entrenched (code, never a knob):** notoriety contributes **zero**
  governance influence; the `engagement × renown` *form* + the eigenvector
  principle; `renown × no-participation = nothing`.
- **Legislated knobs (ordinary law — the declared values):** the scalar
  dials live on the **AppSettings** shape (governance-owned, no code
  defaults) — `renown.decayHalfLives` (esteem fast/fragile, notoriety
  slow/sticky), `renown.contextMultipliers`, `renown.qualityWeight`,
  `renown.receptionValence` (≪ a reaction), `renown.receptionWindowS`.
  Per-**emote** valence is *not* central config — it lives on the emote.

Re-legislating any of these re-scores standing on the next recompute
without touching the log (the raw signal is retained).

## Scope — derive, don't track

One value-function; **per-locality / per-guild standing is a *partition***
of the scope-tagged log, not separate state. Scope is multi-axis —
`locality` (the covering `Locality`'s `getAddress()`, resolved via
`AddressApi`, soft-imported to avoid a static cycle) **and** `groups` (the
objective `Group`s shared by source & subject, via the new
`GroupApi.sharedManagedGroups`). The recompute derives each subject's
materialized scope set from that subject's own events:
`{Compact-wide} ∪ groups ∪ localities`. **Two projections of one
quantity:** governance reads the single Compact-wide roll-up;
NPC/social/disguise read the per-`Group`/per-locality vector.

## The recompute

`recompute()` re-scores every subject from the raw log through the current
value-function into the materialized aggregate, then refreshes the read
cache. It reads **only** the log + AppSettings (+ `SoulApi.all()` for emote
valences) — **never the belief store**. Scoring splits: **reactions** sum
linearly (signed emote valence × signed-decay [esteem half-life positive,
notoriety negative] × context); **receptions** accumulate a decayed,
context-weighted count, then enter log-saturated. The scheduling is a
**real-time `ScheduleApi.recurring`** (`RENOWN_RECOMPUTE_MS`, a code
constant — cadence is mechanism, not a legislated value; decay math still
uses game-time `at` deltas), self-registered at boot, no
`WorldClockRegistry` coupling. Activation = the `RenownLogic` singleton's
presence.

## Sibling of regard, not child

The same reaction fans out to **two independent consumers**: the belief
store updates the reactor's per-pair, scope-less `regard` (overwrite); the
renown log appends a scope-tagged event. The renown recompute never reads
the belief store. (The reaction→regard arrow itself is *deferred* — a raw
reaction has no principled signed regard delta without the value-function,
and `regard` has no recompute to apply one; this build wires only the
renown arrow.)

## Persistence gating

Renown's connection guards go through **`PersistApi`** — a
call-security-decorated chokepoint over the ungated `PersistenceManager`
singleton, introduced with this build and enforced by the **`lint:pm`**
CI gate (`PersistenceManager.get()` is forbidden outside the persistence
framework, the backend, `api/hot-reload`, the facade, and tests). See
[persistence.md](./persistence.md).

## Deferred (seams left open)

Consumers (governance influence, NPC behaviour, disguise/notoriety,
the guard-persuasion resolution where susceptibility + renown + regard
meet); the **eigenvector recursion** (`source` retained → a pure
recompute-time upgrade, no migration); true **engagement-effect** sampling
(the causal Facebook metric — reactions + reception are the v1 proxies;
the open `kind` field absorbs it); **per-institution value overrides** (the
derive-don't-track model never needs them for per-locality *data*); the
reception **debounce**; **log compaction** (decay is a recompute weighting,
so a compaction that drops fully-decayed rows can never corrupt standing);
and the reaction→regard poke.


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

## ⚠ Seeding the log does not move the figure

`RenownApi.renownOf` reads `RenownStanding.cached()` — an in-memory map
warmed at boot from the **`renown` collection**, which holds the
*materialized* standings. It does **not** read `renown_events`.

So writing rows into `renown_events` (as
`packages/server/scripts/seed-standing.ts` does) changes nothing until a
**recompute** folds the log into the standings collection. A bare
restart re-warms the map from a collection the seeding never wrote, and
is therefore a no-op.

This is the trap that made a correctly-seeded character still read
`dormant` through a whole live drive of the S1 build. The same holds for
`participation` and `producer`. `transcripts` and `disposition_events`
are the opposite — they derive on read straight from the log, so seeded
rows are live as soon as the fold runs.
