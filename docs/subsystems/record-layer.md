# The record layer

**What the server remembers for you, and how you get it back.**

Three faces of one subsystem, all behind `RecordApi` (`api/record.ts`)
with its logic singleton at `/obj/api/record`:

- **The frame store** — a per-player rolling window of the frames you
  were actually delivered.
- **`recall`** — one verb over several corpora: your frames, the wiki,
  the forums.
- **The reset policy** — what survives the night, and the job that
  enforces it.

They are one subsystem rather than three because they answer one
question in three tenses: what was I told, can I find it again, and how
long does it last.

## The frame store

### ⭐⭐ One producer, at the delivery seam

The write lives in `Avatar.handleMessage` — **above** the multiplex,
**below** `filterMessage`. That is the only point in the system reached
exactly once per *delivery to a player*:

| Tap point | What goes wrong |
|---|---|
| the socket write | records twice for someone on two devices |
| the scene composer | records frames the recipient's sensorium dropped |
| `Avatar.handleMessage` | ✅ once per delivery, per player |

⚠ It is also reached while the avatar is **linkdead** (the forwarding
loop is a no-op then), which is correct: a frame delivered while you
were disconnected is still a frame you were told, and it is waiting when
you come back.

⚠ **Guests are skipped.** A guest body is reaped when its connection
drops and can never reconnect, so its rows would never be read — and
because every guest is a NEW owner key, the per-owner window that bounds
everyone else would never bound them. That is an unbounded set of small
leaks rather than a bounded one.

### ⚠⚠ The write is off the render hot path

`RecordApi.record` is **synchronous, allocation-cheap and does no I/O**:
it stamps a sequence number and pushes onto an in-memory buffer. A
recurring drain (`record.frames.flushMs`, default 1s) does the Mongo
work, one `save` per row.

The cost, stated rather than hidden: **up to one flush interval is lost
if the process dies.** That is the correct trade for scrollback — the
alternative is an awaited write inside every message fan-out, and this
is the highest-volume write in the system. `AppBootstrap.shutdown`
drains the buffer, so a clean stop loses nothing.

### The sequence number

`seq` is monotonic **per owner** and derived from the wall clock
(`max(now, last + 1)`), so a fresh process starts above anything a
previous one wrote without reading the store first.

⚠ Ordering is **never** by `at`. A scene fans several frames in the same
millisecond and a timestamp sort would shuffle them.

### The window

Retention is a **frame count**, oldest evicted first —
`record.frames.window`, default 2000.

⚠ Deliberately **not** the mailbox model the attestation slate specifies
for clips. The two are different problems: a *clip* is evidence in a
dispute that can take weeks, so it never expires and you delete to make
room; a *frame* is your scrollback, whose value decays fast and whose
volume does not. Giving frames the mailbox model would grow without
bound for every player forever, to serve a case attestation covers
better.

⚠⚠ **The frame store is not a back door to clips or attestation**, and
must not become one. Those stay deferred: a clip is evidence in a
dispute, with a retention model, a chain of custody and a deletion
story this collection deliberately does not have. Anything that needs
those wants the attestation substrate, not a longer window here.

⭐ Eviction is **lazy** and reads its cut point from the store. Trimming
on every flush would cost two indexed operations per active player per
second to delete, on average, one row; letting the store run to
`window + 64` and trimming in one range delete costs the same two
operations per 64 frames. The store's contract is *bounded*, not
*exact*. The cut point is read (`sort seq desc, skip window, limit 1`)
rather than computed, because `seq` is a clock value and `high - window`
would be arithmetic on milliseconds.

### ⚠⚠ Owner-only, structurally

**No read on `RecordApi` takes an owner.** Every read derives its
subject from the acting context, so "read someone else's record" is not
a call that can be written rather than one that is refused at runtime. A
wizard is not an exception because there is nothing to except.

Stating it matters because the frames are not all self-addressed — a
room frame you received names other people, so the store contains
third-party content by construction. The gate is on the **store**, not
on each frame's content, and that is the only tractable place for it.

The one exception is `RecordApi.backfill(owner)`, and it is mechanical
rather than a relaxation: `Avatar.enter` runs outside any command frame,
so there is no acting context to derive from — and only framework files
(`mud/api/**`, `mud/lib/security/**`, `backend/**`) may open one. The
frame is opened inside the Api tier and `recent()` stays the single
owner-derived read.

### ⚠ The client buffer becomes a cache

`ConnectionEstablishedPayload.frameBackfill` carries the window on the
welcome payload. Before it, the client buffer was **the only copy**:
clearing site data destroyed your scrollback and a second device started
empty.

This **changes the meaning of an existing behaviour** rather than adding
one — the same category as Build C's reconnect fix. `clearFrames` on
disconnect is deliberately left in place until the backfill has been
driven live. On a fresh entry the buffer seeds from the backfill; on a
reconnect it keeps what it has and appends only what is missing (deduped
by frame `id`), because frames delivered while linkdead are the whole
reason a reconnect needs the store.

### The row

| Field | |
|---|---|
| `owner` | the player's `playerId` — a durable key, not a `stuffId` |
| `id` | the frame's own id |
| `topic` / `body` / `tags` | verbatim; the client re-renders the MML unchanged |
| `at` | `meta.timestamp` |
| `seq` | the per-owner order key |

⚠ `meta.frameId` and `payload` are **absent by design**. `frameId` is
stamped per-Interactive, so storing one would make the record a
statement about a socket rather than about the player; `payload` is
per-topic structured data whose consumers re-resolve on connect anyway,
and carrying it would make the highest-volume write unbounded in row
size.

Indexes: `{owner: 1, seq: -1}` serves the backfill read, the `recall`
scan **and** the eviction delete; `{owner: 1, body: 'text'}` is the
search index.

## `recall`

**One verb, a scope argument, three corpora.** Scope is a named argument
rather than a verb per corpus, so a fourth corpus is an entry rather
than a verb.

⚠ `search` is taken — it is the in-world perception verb (looking around
a room with your own eyes). The two are not variations on each other.

⚠⚠ **`recall` must not drift toward a client-side filter box.** A box
that filtered the local buffer would look identical, work on one device,
and quietly break the axiom that the client owns zero command semantics.
Every invocation is a real command on the wire.

### The search

Mongo text indexes and a projection. No external service, no embedding
model, no ranking design — the requirement is *find the thing you
half-remember* and the corpora are small.

⚠ `$text` is **stemmed and language-aware**, so it misses the substring
searches players actually type (`recall thermo`). Every scope runs
`$text` first and falls back to a bounded case-insensitive scan when it
returns nothing. That is not a second search engine — it is the same
query with the index turned off.

⚠ Excerpts are **quoted, not re-rendered**. `Mml.flatten` strips the
stored body to plain text before it is escaped into the readout; pasting
a stored body's live markup into a fresh frame would let it act in a
context that never composed it.

⚠ A soft-deleted wiki page does not resolve. A search that surfaced one
would be a back door around the delete.

### ⚠⚠ One text index per collection

Mongo allows exactly one, and `createIndex` on a second *shape* throws
rather than merging. `PersistenceManager.ensureTextIndex` drops a
differently-shaped one and rebuilds — safe in a way it would not be for
a unique or a TTL index, because a text index is a pure derived
structure over data that is still there.

⭐ Found by driving. A dev database carrying an older `{body:'text'}`
rejected the compound `{owner:1, body:'text'}`, the server booted
anyway, and `recall` fell back to its bounded scan **forever**. Worse:
one failing `createIndex` aborted every index declared after it, so the
wiki index, the forum index and every sandbox partial index went with
it. The three search indexes and the sandbox partials now sit in their
own `try`, and the outer catch says what a failure costs.

## The nightly reset

⚠⚠ **Destructive by design, and there is no reflog.**

### The survivors list

Decided by the user, 2026-08-14: the reset removes all **player state**
except `documents` rows with `kind: 'release'` — the press releases the
front door's press room displays. Accounts included.

That list is sharp *because* it is short: `RELEASE_DOCUMENT_KIND`
already exists as a discriminator, so the survivor predicate is one
equality rather than a policy table.

### ⚠ Seeded world content is not player state

The seeder and the content-pack installer are **insert-only and run at
boot**. The job does not restart the process, so anything they populate
is `keep`: wiping it would empty the world until somebody rebooted, and
*a wipe that empties the world is a wipe that broke the game.* This is
the one place "wipe everything", read literally, is wrong.

⚠ The knowing cost: CMS-authored templates live in `domain` beside the
seeds and therefore survive too. No discriminator separates them
(`sourcePack` marks pack rows, not authored ones), and an empty world
every morning is worse. Recorded so it is a decision rather than a
surprise.

`app_settings` is kept for a sharper reason still: the job reads its own
arming **and** its own reset policy from there, so wiping it would
disarm the job and silently retract the front door's notice on the first
run. `world_state` is kept because a calendar that restarts at zero
every morning makes every in-world date a lie.

### The table is total

`lib/persistence/ResetPolicy.ts` is a `Record<Collections,
ResetDisposition>`, so **a new collection cannot ship without a
decision** — the same trick `COLLECTION_POLICIES` uses, failing closed
at build time rather than at an audit. Every survivor states its reason
at the site, and the suite asserts that it does.

### The three safeguards

- **Dry-run is the default**, and it counts on the *same predicate* the
  enforce path deletes on. Counting a different query is how a dry run
  reassures you about a wipe that then takes something else.
- **A loud log**, per collection, on every run.
- **A test that asserts both directions**: the survivor survives AND a
  representative row from every wiped collection does not. A wipe that
  quietly takes one category too many and one that quietly leaves one
  behind are indistinguishable from working.

### Arming

| Setting | |
|---|---|
| `world.reset.mode` | absent → nothing installed; `dry-run` → logs; `enforce` → deletes |
| `world.reset.intervalMs` | cadence, default 24h |
| `world.resetPolicy` | the PROSE the front door prints |

⚠⚠ `RecordApi.boot` **refuses to arm enforcing while `world.resetPolicy`
is unset.** A server that wipes without printing the notice is a server
whose front page lies about what happens to your work.

### ⚠⚠ The front door's persistence copy

The sign-in panel used to read *"Your character, your record, and
everything you build persist to an account."* Under a nightly total wipe
that is the same class of defect as *"Sign in to save"* — copy the
player **acts on**, believing their work is banked. It was replaced in
the commit that shipped the job, not after it, and a test asserts the
old sentence is **absent** (asserting the new one is present would pass
just as happily with both there).

### Founder access is part of the job

Two traps, both recorded before they were hit:

1. **The wipe takes `groups`**, which carries the SYSTEM groups (`core`,
   `wizards`, `archwizards`, `streamers`). They are minted in code by
   `AccessRegistry.postRegister`, not by a seed file, and that seeding is
   boot-only — so without a re-seed the world comes back with no `core`
   group, every resource-targeted `can` read failing closed, and no fix
   short of a restart. The job re-seeds them, dropping the cached refs
   first (they point at rows the wipe just deleted).

2. **`WIZARD_PLAYER_IDS` names CHARACTERS**, and the wipe takes the
   characters — so a grant keyed on `playerId` is void by the next
   morning, and the env seed is boot-only on top of that.

   ⭐⭐ **The answer is not a second credential — it is the office.**
   Whoever holds the Prime Minister's seat is a wizard and an
   archwizard, derived on every check (`AccessRegistry.holdsPrimeMinister`).
   The world ships with **no wizards at all**; there is exactly one
   credential anywhere in the system — the founder's provider id, read
   by `OfficeRegistry` — and all it does is make the founder the DEFAULT
   HOLDER until somebody is seated explicitly. A reset cannot sever
   that, because it was never a stored grant.

   ⚠ An earlier pass added `WIZARD_EMAILS` / `ARCHWIZARD_EMAILS`,
   granting group membership by account email at `Avatar.enter`. It is
   **removed**: it was a second credential surface beside the founder
   id, and a stored grant survives the handoff that was supposed to end
   it — hand the office on and the old holder silently keeps code trust.
   *Check offices, never the founder.*

## Configuration

| Key | Default | |
|---|---|---|
| `record.frames.window` | 2000 | frames kept per player |
| `record.frames.flushMs` | 1000 | drain cadence |
| `record.recall.limit` | 20 | hits per scope |
| `world.reset.mode` | absent | `dry-run` \| `enforce` |
| `world.reset.intervalMs` | 86400000 | |
| `world.resetPolicy` | absent | the front door's words |


## Cross-references

- [persistence.md](./persistence.md) — the collection vocabulary
- [messaging.md](./messaging.md) — where a frame comes from
- [client-shell.md](./client-shell.md) — the buffer that is now a cache
- [press.md](./press.md) — `RELEASE_DOCUMENT_KIND`, the one survivor
- [connection.md](./connection.md) — the welcome payload

## ⭐ `recall --scope` is wired to the wiki card

The wiki card's search box sends `recall --scope wiki <terms>` — a real
command, previewed exactly as sent, with the results landing in the
transcript like every other answer.

⚠ It shipped hatched, reading *there is no search port yet*, citing an
audit that was already stale: `recall --scope wiki` had merged **before**
the wave that wrote the hatch. The hatch was written from a table rather
than from the tree, which is the failure mode a stale hatch always is —
it tells a reader the surface does not exist when it does. A test now
greps the client SOURCE for the retired string.
