# Instrumentation — the reading ladder

**What a character can find out about the world, and what it costs
them.** One flat verb pair, one row per fact, and a ladder of routes
underneath: the trained eye, the carried instrument, the bench.

> ⭐⭐⭐ **Competence resolves DETAIL. It never resolves ACCESS.**
>
> Every channel is open to everybody. What training and an instrument
> buy is how **narrow** the answer is. Nothing here is ever refused for
> being untrained — where a reading is out of reach it is a **route**
> you are missing, and the refusal names it.

## The shape

| | |
|---|---|
| `analyze <channel> [subject] [with <tool>]` | the **free** rung — by eye, banded by competence |
| `measure <channel> [subject] [with <tool>]` | the **instrumented** rung — a figure with an honest bracket |
| `readings [channel]` | the ladder itself: every channel, every rung, whether it is open to you, what it settles and what not knowing costs |
| `sample [subject]` | take a real piece, stamped with where it came from |
| `assay [samples] [at <bench>]` | a bench consumes samples over game-time and hands back a paper |

⚠ `trace address` and `trace atmosphere` are **not** readings. They
answer *where did the engine get that value from*, which is a question
about the simulation, so they live on a free `system` verb beside
`affordances` and `errors`.

## ⭐⭐ A channel is a row any pack can ship

The whole registration story for a fact the world can tell you:

```
packages/content/<pack>/content/<root>/idea/reading/<channel>.yaml
  class: /<root>/idea/reading/<Channel>Reading
```

Install the pack, get the channel; remove it, lose it. **No kernel list,
no stanza in a platform view, no boot-sequencer line.** `trade-mining`
brings `strike`, `dip`, `ground`, `grade`; `trade-farming` brings
`texture`, `acidity`, `soil`; `water` brings `power` and `water`.
31 channels ship today across the platform and seven packs.

`ReadingCatalogue` (`platform/idea/ReadingCatalogue.ts`) stands every row
up as a live singleton and indexes it by the token a player types. It
selects by **template-path infix** (`/idea/reading/`) across every root
and keeps a row whose class extends `Reading` wherever it lives — never
an allowlist of roots, which is exactly what lets a pack's own channel
qualify.

⚠⚠ **It warms lazily as well as at `postRegister`.** The verbs resolve a
channel on *every* dispatch, so the dispatch path must not depend on
boot order. The inert-reference-Idea trap has bitten this repo three
times: a roster nothing stands up reads null forever on a fresh process
and every consumer silently answers *"no such thing"*.

⭐ **There is no `InstrumentApi`.** Callers resolve the catalogue and ask
it — `StuffApi.singleton<ReadingCatalogue>(READING_CATALOGUE_PATH)` then
`warmed(channel)`. `singleton` *is* the get-or-create, so boot-order
independence is a property of the call rather than of a tier. A
two-method Api over a catalogue held no logic and could never grow, and
both tiers were deleted at review.

## The `Reading` class

`lib/instrument/Reading.ts` — an abstract `Idea`, one singleton per
channel, carrying both the row's data and the rungs' behaviour. **A
reading is an act performed by a channel, not a function of a
subsystem**, which is why everything about it lives here rather than on
an Api.

| field | what it says |
|---|---|
| `channel` | the token a player types |
| `kind` · `scope` | what sort of fact, and what it is read ABOUT (`subject` · `here` · `self`) |
| `discipline` | which Discipline bands the eye rung (empty ⇒ the same for everybody) |
| `instrument` · `instrumentNoun` | the capability the `measure` rung wants, and **what to call it in a refusal** |
| `handTool` | a lens that raises the EYE rung's ceiling |
| `bench` | the capability a bench rung wants |
| `eyeCeiling` | how good the free rung can ever get |
| `subjectRequires` | mixins the subject must compose |
| `improves` · `stakes` | ⭐ what knowing it settles, and what not knowing costs — every row authors both, and `readings <channel>` prints them |

Rungs are `@hook`s a channel overrides: `analyze`, `measure`,
`benchRead`. `truth()` is the engine's own ungated read; `observe()` is
what a given reader got out of it.

### The honest bracket

⭐⭐ **The roll decides what you can TELL, never what the world IS.** A
reading is seeded, not drawn — the same reader, the same subject, the
same answer — and the bracket widens with the reader's band:

```
untrained ±8%   novice ±5%   competent ±2.5%   proficient ±1.2%   expert ±0.5%
```

An instrument raises a **ceiling**; competence is what actually realizes
it. The effective band is `min(bandOf(actor), ceilingOf(instrument))`,
so a novice with a good dial reads better than a novice without one and
worse than an expert with the same dial.

⚠ **The tag comes off the OBSERVATION, not the truth.** A drive printed
`153.01 K ± 147.5 K (warm)` — the number was what the reader saw and the
word was what the world knew, and one sentence contradicted itself. A
reader who misreads the dial calls the room what the dial said.

See uncertainty.md for which provenances of
randomness are legal; this is the **epistemic** one.

## Samples, benches and the paper

`SampledMixin` (`lib/instrument/Sampled.ts`) stamps three inert fields —
`sampledAt`, `sampledBy`, `sampledOn`. ⭐ **Provenance is a historical
claim, not a live reference**: `sampledAt` is a path *string* and is
never resolved back to an object. Consumers group by the string.

A bench (`AssayBench`, `trade-mining`) is the water mill's shape applied
to a reading: you leave samples on it, it works through them over
game-time, and it **holds nothing of yours** — no engagement, no lease.
Walk out, log off, come back; the papers are on the bench. The batch is
**amortized** (`setup + n×per`, not `n×(setup + per)`), which is the
whole economics of the trip.

⚠⚠ **A completion runs OUTSIDE the dispatch frame.** `WorldClockApi.after`
callbacks are module-level functions in a *fresh root* execution frame,
so a `FromModule` gate on the bench read is unsatisfiable by
construction — and the resulting `SecurityError` is an unhandled
rejection in a timer callback, which **kills the process**. The bench
read is ungated and each sample has its own `try/catch`.

`ReadingRecord` is the paper. ⚠ `inscribe` writes the **`longDescription`
field**, not just an overridden getter: `look` and the inspection card
render the stored field and consult no getter.

## ⚠⚠ Two things that fail closed and silent

**The article defect.** Any object arg a player may put an article in
front of needs `greedy: true` — **including plural and prepositional
args**. Without it `assay the ore` binds the literal token `the`, and
`measure temperature the kettle` trips *"too many arguments"* and dies as
an unknown shape with everything upstream of it correct.

**A required arg with no default.** `assay` shipped `samples` as
`required: true`, so the bare form could never bind and a player standing
at a bench was told *"That doesn't match any known command shape"* — the
parser, about the word, for a verb the room affords. The controller's own
refusal was unreachable from the moment it was written. ⭐ **The refusal
IS the progression UI**: if something lifts a limit, the verb has to
exist so you can be told.

## Prose is a surface, and it is tested by reading it

⭐⭐ Every finding of the live browser drive was **prose a player reads**,
and that is the class a wire assertion structurally cannot see: it
matches a substring and is blind to the rest of the sentence. A
checkpoint matching `/compass/` passed while the line read
*It wants \\"a surveyor's compass or a miner's dial".*

`lint:authored-prose` (ceiling 0) now refuses an authored YAML scalar
that opens with an escaped quote. The rest is: **read the words**.

## Cross-references

- measurement.md — what the platform may count, and
  the **no-gauge reading rules** this subsystem obeys
- uncertainty.md — the four provenances; readings are
  epistemic and seeded
- [advancement.md](./advancement.md) — the Disciplines that band the eye
  rung
- [mining.md](./mining.md) · [soil.md](./soil.md) ·
  [watershed.md](./watershed.md) — packs that ship channels
- [command-routing.md](./command-routing.md) — the binder, and the
  `greedy`/`required` rules the article defect turns on
