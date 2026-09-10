# Display — screens: what a display shows, and who sees it

The display substrate (libations D12): a **tablet, a wall TV and the
terminal's departures board are one thing** — a `Thing` that shows one
source to everyone who can see it. Output is optical; the difference
between the three is data.

Source: `lib/display/Display.ts` (`DisplayMixin`),
`platform/thing/{Tablet,Screen,Remote}.ts`,
`world/common/tpa/TpaTerminal.ts` (composes the mixin). Tests:
`lib/display/__tests__/Display.test.ts`.

## The rule that governs everything here

> **A display confers no money authority.** A thief holding the house
> tablet reads the stock sheet — the sheet is what the screen shows. What
> the thief lacks is the seat: `wallet use house` refuses them, `buy`
> settles from their own account. Money authority is only ever the
> wallet's, and the wallet's is the position's ([employment.md](./employment.md),
> [banking.md](./banking.md)).

The screen is a *thing* in the room. Verbs are physical acts — `watch …
on tv`, `house stock` with the tablet in hand — and the operations they
drive are apps on a display, never a new verb only a machine could run.

## `DisplayMixin`

Persistent + authorable fields (the row decides which screen it is):

| field | values | meaning |
|---|---|---|
| `pairing` | `remote` · `held` · `staff` · `open` | who may DRIVE it (below) |
| `shows` | a list of `DisplayKind` | which kinds it accepts (`acceptsSource`); defaults to **all three** |
| `principal` | a Business template path or `''` | the `staff` policy's "signed in as" |
| `remote` | a `Remote` row's template path or `''` | the `remote` policy's paired thing |

Runtime only: `_showing: DisplaySource | null` — **a screen is dark on
boot**; nothing persists what it showed. `_setShowing` is gated
~~`FromTemplate('/platform/idea/api/display')`~~ — the gate is gone with the Api; only the mixin
writes it, because a write without the projection below would leave
every viewer's screen stale.

### ⭐ The three arms — `DisplayKind`

The first cut of this mixin modelled *the screen*, *who drives it* and
*what is showing*, and did **not** model **how the client renders it**.
So there were two hardcoded manifestation paths and no third, and
`sourcePolicy: 'any' | 'cards' | 'streams'` straddled the gap — a
PERMISSION field whose values were rendering kinds. The founder's model
is the right one: a display is any screen anywhere, and its contents
manifest three ways, which are three *client components*.

```ts
type DisplayKind = 'video' | 'card' | 'prose';

type DisplaySource =
  | { kind: 'video'; target: WatchTarget; label: string }
  | { kind: 'card'; cardId: CardId; subjectId?: string; key: string; prose?: Mml }
  | { kind: 'prose'; body: Mml };
```

- **video** — an embed the client renders ([streaming.md](./streaming.md)).
  Live vs recorded is a property of the CONTENT; organising content into
  networks, feeds or a guide is a later **addressing** layer that changes
  nothing here. *A stream is a transport (Twitch, YouTube); video is the
  manifestation* — which is why the kind is not called `stream`.
- **card** — an app: controls and feedback, on the card rail
  ([card-surface.md](./card-surface.md)). The `stock` sheet.
- **prose** — ordinary game text, read off the screen.

The kind is **carried, explicit and total**: `project` switches
exhaustively, so a fourth arm is a compile error rather than a silent
fall-through. A screen no longer declares *"I do cards"* — it shows
content, and the content knows how it manifests.

`shows: DisplayKind[]` is what remains of `sourcePolicy`: an honest
policy OVER kinds. It defaults to **all three** rather than to an `'any'`
sentinel — a sentinel has to be taught about every new kind, which is the
central-list failure mode ([antipatterns.md](../antipatterns.md)).

### ⭐⭐ Prose has no projection, and that is the finding

`readScreen(viewer): Promise<Mml | null>` is the whole prose arm: **what
this viewer reads off this screen right now.** The default renders the
showing source — a prose body verbatim, a video or card as the one-line
"Showing: …" that `look` already gave — and a host whose board is
*computed* rather than driven overrides it.

The other two arms push because the client holds a component that must be
told. Prose is text the client already renders, and what a screen says is
**read off it**, at look time, per viewer. Pushing prose would spam the
room on every refresh *and* would flatten a per-viewer board into one
shared payload. So the prose arm is **thinner than a mechanism**: it is
the display declining to use one, and `LookController` calling
`readScreen` is the entire wiring.

## Who drives — the pairing policies (`display.mayDrive(actor)`)

| pairing | drives it iff |
|---|---|
| `held` | the actor carries it (inventory, any depth) |
| `remote` | the actor carries an instance of the row `remote` names — the pairing is the SCREEN's field; a `Remote` is a plain thing |
| `staff` | `EmploymentApi.holdsPosition(actor, principal)` or `isProprietorOf` — the house tablet's "signed in as" |
| `open` | `PerceptionApi.canReach(actor, display)` — the terminal's board |

### The modem is a predicate on the driver

`AetherMixin` is `_augmentGated`: it is active only when a slot augment
or a species confers it, so a slot-less Thing composing it would never
be active — and a display hosts no updates. So the screen composes **no**
aether mixin. Driving *by mind* is `MixinApi.isActive(driver,
'AetherMixin')` checked on the driver, and it works from anywhere the
display exists; driving *by hand* needs reach. (Plan deviation 4 from
the requirements' wording; the five D12 behaviours hold.)

### The implicit-screen ladder (`CommandController.resolveScreen`)

Returns `{ display, mode: 'hand' | 'mind' } | null`:

1. **Held** — a screen in your hand is yours to drive **whatever its
   pairing**. This rung is deliberately unconditional: it is what lets
   the thief read the sheet on a `staff`-paired tablet. The seat is
   checked where money moves, never here.
2. **Paired and in sight** — a screen in the actor's room that
   `mayDrive` admits.
3. **Paired anywhere, by mind** — with an active `AetherMixin`, any
   `held`/`remote`/`staff` screen `mayDrive` admits (`open` screens are
   never driven remotely). The driver by mind **sees nothing** of what
   they show — the projection is to the screen's room, not to them.

A command that needs a screen and finds none declines `no-display`
("you'd need a screen").

## Who sees — the projection rule (`display.show(source)`)

> *The display you can see shows X.*

`show(display, source)` sets `_showing`, then projects to **every
viewer**: a `HasInteractive` Stuff with at least one Interactive
attached, in the display's resting room, passing
`PerceptionApi.perceives(viewer, display)` (that predicate is a
concealment gate only, so the room check is explicit). Derived from the
world on every projection — never from the connection registry.

- A **video** source writes each viewer's `cockpit.watch` clientState —
  the same key the personal `watch` writes — with a `display: { stuffId,
  label }` marker, and pushes it. No new wire message.
- A **card** source is `CardApi.push`ed to each of the viewer's
  Interactives with the display's presentation as the card `title`.
  That is the card rail's ONE birth path: the onlooker's card is a fact
  the server pushes, not one the client infers. (The `card-birth-path`
  test's mint set records `Display.ts:push`; when a display is in
  play, `show` is the birth path and the driver's own card is one of the
  projected viewers' — no double push.)
- A **prose** source projects **nothing** — see above. It is read.

`clear(display)` darkens it and clears every projected viewer's
`cockpit.watch` that names it. `refresh(display)` re-projects the current
source. `viewersOf(display)` is the derived viewer list.

### Arrival and departure — `viewer.refreshDisplays()`

One hook, called from `Mobile.traverse` and `Mobile.teleport` after the
move: project every lit display the viewer now sees, and if their
`cockpit.watch` names a display they no longer see, clear it. Walking
into a room with a lit TV shows what it shows; walking out, the shared
embed leaves with you. A personal watch (no `display` marker) is untouched.

## The instances — two shipped, one waiting

| | class | pairing | `shows` | row |
|---|---|---|---|---|
| **house tablet** | `/platform/thing/Tablet` (`Display(Detailed(Thing))`, portable) | `staff` | `[card]` | `/trade/hospitality/thing/house-tablet` (`principal: ''`); the lounge's `/world/lounge/thing/house-tablet` sets `principal: /world/lounge/idea/business` |
| **a wall TV + remote** | `/platform/thing/Screen` (`Display(PostRegistration(Fixture(Detailed(Thing))))` — self-seats via `seatIn`, and defaults `fixedInPlace: true`: **mounted, not immovable** — no agent pockets it, while a remodel or a `place` still moves it. It shipped as a `canMove` veto, which was both too low a level and too rigid for a row that stands one on a counter; see [spatial.md](./spatial.md)) + `/platform/thing/Remote` (`Detailed(Thing)`; the row authors `keywords: [remote]`) | `remote` | all three | **no row ships.** The classes, the `remote` policy and `watch … on <screen>` are proven on synthetic fixtures (`Display.test.ts`, `WatchController.test.ts`); the first row is the LOUNGE's sports booth (lounge-slate § *Themed booths*), not Dave's Bar — the bar is where soft skills get evidenced, not where you stare at a screen |
| **the terminal** | `TpaTerminal` composes the mixin; the constructor sets `open` / `['prose']`, and overrides `readScreen` — its board is COMPUTED, not driven | `open` | `[prose]` | the lounge terminal, unchanged |

## The verbs that drive a display

- **`house stock`** ([employment.md](./employment.md)): `resolveFor(giver)`
  → `no-display` if none → emits the sheet as prose and `show`s the
  `stock` card (`Cards.ts`: an `mql` source over
  `reachable:[mixin.BulkableMixin]`, live) through the screen. When the
  screen's `principal` is set, the house read is the screen's (the
  signed-in tablet); otherwise the giver's seat. `house par` — a write —
  stays seat-gated.
- **`watch <target> on <screen>`** / **`watch off on <screen>`**
  ([streaming.md](./streaming.md)): the `on` object arg resolves in
  `peers` scope with `requires: DisplayMixin`; `mayDrive` decides;
  `show`/`clear` project. Personal `watch` (no `on`) is unchanged.
- **`teleport`** (bare, at a terminal): the departures board renders for
  the reader, and for the reader only. ⚠ It formerly `show`ed a `subject`
  card carrying the board's prose to everyone in reach — which was the
  symptom that proved the missing arm (a board is plainly not an app with
  controls and feedback; it shipped as a card because card was the only
  non-video arm) **and a real defect**: `renderDepartures(viewer)`
  annotates each route against the READER's own travel credential ("— not
  yet registered"), so the whole room got one traveller's registration
  state — wrong for all of them, and nobody else's business. The board is
  now `TpaTerminal.readScreen(viewer)`, which also makes
  **`look <terminal>` the way you read the board**.

## What the client changed

**Nothing, in either wave** — and that is a result, not an omission. The
manifestation kind decides which client component renders the content,
and each arm already reaches its own: video the embed, card the rail,
prose the text stream the client has always had. `StreamEmbed` reads the
`display` marker off the target it is handed; it does not infer it. The
"one projection, one wire shape" consolidation the slate raises stays
open, and would be its own build.

From the D12 wave, a caption. `@saxonberg/types` `WatchTarget` gained
`display?: { stuffId: string; label: string }`. `StreamEmbed.tsx` renders
"on <label> — whoever holds the remote switches it off" under a shared
embed (`data-testid="display-caption"`) and hides the personal
empty-state copy; the iframe path is identical. Two RTL tests.

## Non-goals (v1)

- A `channel` list / guide on the TV — the remote drives `watch … on tv`
  with the stream grammar; a channel vocabulary is the lounge's later
  build.
- Persisting what a screen shows across a reboot — dark on boot, by
  design.
- A screen as an aether host (see the modem rule above).
- Driving an `open` screen by mind.
- Multiple simultaneous sources per screen (`showing` is one slot; a
  screen running a card *and* a ticker is a real thing and would want
  more — the slate's open question 3).
- A network / channel / guide layer above `video` — **addressing**, not
  rendering, and deliberately deferred.
- Driver policy off the closed `pairing` enum. Real, and the same
  central-list finding as three other fixes in this review; it is an
  ACCESS change ([access.md](./access.md)) and gets its own commit.

## Cross-references

[streaming.md](./streaming.md) (the embed key, `watch … on`),
[card-surface.md](./card-surface.md) (the one birth path),
[employment.md](./employment.md) (`house`, the seat),
[fasttravel.md](./fasttravel.md) (the terminal), [augmentation.md](./augmentation.md)
(`AetherMixin` activity),
display-manifestation-slate.md
(the three-arm model and what it deferred).

## ⭐ There is no `DisplayApi` — a display drives itself

The first cut of this subsystem shipped a seven-method `DisplayApi` +
`DisplayLogic` pair. It was built by **pattern-matching the shape of
other subsystems** rather than by asking whether display needs
orchestration, and CLAUDE.md had already answered that:

> An Api method exists to **orchestrate** — movement, lifecycle,
> cross-object dispatch. A read or mutation that belongs to ONE object
> lives on that object… Don't add a thin Api wrapper around a single
> object method.

`DisplayApi.mayDrive(actor, display)` was the tell: `FooApi.f(fooStuff)`
where the object should simply answer for itself. Five of the seven
methods were about one screen's own state and moved onto the mixin —
`mayDrive`, `isCarriedBy`, `viewersOf`, `show`, `clear`, `refresh`.

⚠ **Fanning out to N viewers is not the test.** `victim.afflict(...)`
dispatches across half the engine and still lives on the victim; the
test is *whose state is this fundamentally about*. `show` mutates the
screen and then tells the room — that is a screen doing its job.

The two leftovers went to homes that already existed, because two
methods do not justify a namespace:

| leftover | home | why |
|---|---|---|
| the arrival/departure hook | **`HasInteractive.refreshDisplays()`** | it re-syncs the *viewer's* client state, and this mixin owns that state — it declares the `cockpit.watch` key itself |
| the implicit-screen ladder | **`CommandController.resolveScreen()`** | it is the fallback for a command that named no screen; the explicit form already resolves through the ordinary arg machinery |

`_setShowing`'s `FromTemplate` gate existed only because the Api was the
one legitimate writer. With the writer inside the class it collapsed
into an ordinary private field.

