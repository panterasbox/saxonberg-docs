# Client shell — frame, start screen, guest & connection-loss

> **This documents a *client* subsystem.** Most of `docs/subsystems/`
> describes the server engine; this one covers the React client's
> top-level shell — the frame primitives, the plain-UI start screen, the
> anonymous-guest path, the portrait, and the connection-loss state
> machine. Its server-side touchpoints (the guest principal, the portrait
> resolver, the don't-flush seam) are noted where they cross the wire.
> Seeded by client-shell-slate.md;
> built per `docs/requirements/client-shell-frame-requirements.md`.

## The top bar

`components/frame/Frame.tsx`. One row, rendered once above the layout
registry, so it is present in every mode:

```
[seal] [ConnectionChip] │ [AccountMenu] │ [──── Shelf (flex:1, wraps) ────] [Views] [Settings]
```

**Identity and connection are anchored LEFT, together**, because they
are *the two things that must be true at a glance whatever else was
removed* — you are logged in as somebody, and the bus is alive. The
shelf is everything else, and everything else is negotiable.

⭐ **Identity and connection cannot be unpinned — not by a rule, but
because they are not shelf rows at all.** `cockpit shelf unpin identity`
refuses with *unknown shelf row*, which is a stronger guarantee than a
protection rule somebody could edit away.

The components:

- **`ConnectionChip`** — the always-visible connection surface, and the
  popover behind it. See § *The connection popover* below.
- **`ConnectionIndicator`** — unchanged, and **composed rather than
  grown**. It is silent when `connected` (renders `null`), which is
  exactly why it could not become an always-visible chip; the chip owns
  the healthy state and *delegates* the unhealthy vocabulary to it.
  Three things follow by construction: the indicator's own test passes
  unmodified, there is exactly ONE rendering of "Reconnecting…" /
  "Disconnected" in the tree, and no dot is drawn twice.
- **`AccountMenu`** — a dropdown off the identity label (portrait +
  name), state-polymorphic over the connected identity: a real character
  gets *Switch character* (a reconnect → roster, account stays signed
  in) + *Sign out*; a guest gets *Sign in to start a character* +
  *Sign out*. ⚠⚠ That item used to read *"Sign in to save"*, which
  promised the one thing a guest session explicitly does not get — the
  front door states a guest keeps nothing and nobody can find them
  again, and there is no conversion path. Copy that implies the session
  carries over is worse than a wrong figure, because the player **acts**
  on it: they keep going, believing the work is banked. The two
  exits are distinct — leaving the world ≠ ending the account.
- **`Portrait`** — the small avatar image; renders `player.portraitUrl`
  with an initials fallback when the URL is empty (the server's "no
  image" sentinel) or fails to load. No broken-image icon ever shows.
- **`Shelf`** — the widget shelf. Its own section below.
- **`ViewsMenu` + Settings** — both **survive** the rebuild. Neither is
  named in the honest-chrome requirements, and migrating the frame off
  `cockpit.layout` is an explicit Wave 4 non-goal, so a "full rebuild"
  that dropped them would have regressed two shipped surfaces.
- **The seal** (`components/frame/Seal.tsx`) — the real Saxonberg mark,
  **applied**: a white rim, a struck red field with two inset rules, and
  the three Borromean rings. Source of truth is `docs/brand/` (⚠ which
  currently lives on the `design/balance-machine` branch, not master);
  the artwork is inlined, which is what the brand README says to do.

  ⚠⚠ **The rim is the flag rule, not decoration.** *Red never touches
  blue; white is the separator*, and this bar's ground is navy. This is
  the surface `--sx-red` was reserved for — the official colour is
  usable at its 2.66:1 **because** it carries white separation, so it
  has to actually carry it. A first cut drew a red block with a red
  border straight onto the navy, which is precisely what the rule
  forbids; `Seal.test.tsx` now asserts the rim exists, spans the full
  radius, and is painted beneath an inset field. `tokens.color.danger`
  still resolves to `ember`, so red stays out of the alert vocabulary.

  ⚠ **The source SVG does not carry the rim** — both its white rules are
  inset, leaving bare red at the outer edge. A white disc is added with
  the artwork scaled to `0.949` inside it, matching what
  panterasbox-web's `Marks.js` does. If the source file grows its own
  rim, drop the disc and the scale.

  ⭐ **The casing is structural.** Each ring carries a field-coloured
  stroke beneath its ink stroke; that is what interrupts the ring
  passing underneath. Drop it and three same-coloured rings stack, no
  crossing reads, and the Borromean claim — no two linked, yet all three
  hold — becomes false. Editing the paths means keeping both crossings
  of a given pair the same way round.

  ⚠ Colour reaches the mark through **CSS rules on classes**, never SVG
  presentation attributes: `tokens.color.*` are `var(--sx-…)` strings,
  and a `var()` in `fill="…"` does not resolve — it silently paints
  nothing. This is the hazard `tokens.ts` warns about, in the one place
  it actually bites.

⚠ **The notification bell in the reference art is not built, not
hatched, and not placeholdered**, and `Frame.test.tsx` asserts its
absence. What belongs in a notification tray is whatever the receiver
*said* they wanted — which wants `NotifyPolicy` / `NotifyRule` read
first — so a stub would be an interface promising a model that does not
exist. That is the same failure the honesty convention prevents, one
level up: not a fake figure but a fake capability.

⚠ **Identity sits left, against one sentence of the requirements.** The
requirements say "the account menu at the right"; the reference art puts
the `who` chip at the left, beside connection. `AccountMenu` is one
component that is *both* the identity chip and its dropdown, so it
cannot be in two places. The left reading wins on the requirements' own
stronger sentence (the "two things that must be true at a glance"). If a
split was intended — identity chip left, account actions right — that is
two components and a second identity rendering, and it is a design call.

**Shared primitives, not a shared `Frame` across surfaces** — the start
screen composes the same primitives into its own full-screen layout; a
single cross-surface `Frame` wrapper is deferred until a third surface
(CMS, public) makes the real commonality visible.

## ⭐⭐ The server owns what is shown; the viewport owns how it is disclosed

The one line that decides every *is this client or server* question in
the chrome.

`cockpit.shelf` is **identical on both form factors** — same rows, same
order, same `cockpit shelf` verb, same catalogue. What a viewport
decides is only *disclosure*: whether the shelf renders inline in the
bar (desktop, wrapping) or behind a pull-down (mobile, two-up).

⚠ **Mobile is a viewport fact and must never become a server key.** The
server cannot know a viewport, so a `cockpit.formFactor` would be a fake
fact — the honesty failure one level up from a fake figure, and the same
one that cut the read-only indicator below. `nothing-is-pure-client`
governs **command semantics and affordances**, and none of those move:
every affordance still sends a server command and the client still owns
zero command semantics.

Detection is `lib/style/useIsCompact.ts` over `matchMedia`, against the
single named token `tokens.breakpoint.compact` (**760px** — above phone
landscape, below tablet portrait) so it cannot drift between components.

⭐ **A JS switch, not a CSS-only reflow.** The mobile bar is not the
desktop bar rearranged; it is a different composition — two rows, a
pull-down, no status bar. CSS-only means rendering **both** and hiding
one, which puts two `StatusBar`s in the DOM permanently and makes the
requirement *"exactly one status bar renders above the breakpoint"*
unassertable by construction. That
`__tests__/chromeComposition.test.tsx` can make that assertion at all is
the argument for the switch.

⚠ The hook **subscribes** to `change` rather than reading once. Dragging
a desktop window narrow is the cheapest way anybody will test this, and
a one-shot read would make exactly that not work — the chrome would stay
whatever it was when the tab opened, which reads as a broken layout
rather than a stale hook. Where `matchMedia` does not exist at all it
resolves to *not compact*, the composition that has been shipping.

⚠ **`viewport-fit=cover` in `index.html` is part of this, not a
detail.** Without it `env(safe-area-inset-*)` resolves to `0px` and
every safe-area rule renders faithfully to nothing. The failure is
**silent** — the padding is there, it is just always zero — so the e2e
spec asserts both halves separately: that the bar's rule references the
inset, and that the attribute enabling it is present. Either alone
passes while the feature is dead.

⚠ **Scope: the in-world phase only.** `CharGenStage` keeps its own
`StatusBar` and its three ad-hoc breakpoints (640/520). Mobile intake is
Wave 2 with its own art, and reflowing it here would be building half of
that wave blind. The named token deliberately does *not* match those
numbers — they are a different question, and an accidental match would
make two unrelated decisions look like one.

### ⚠⚠ The shell is clamped to the viewport, and that is load-bearing

`WorldLayout` puts a fixed `22rem` rail beside the terminal, so at a
390px viewport its content wants ~698px. Redesigning the mobile play
surface is an explicit **Wave 4** non-goal — but leaving the *document*
that wide is not survivable, and the reason is not tidiness:

⭐⭐ **Under the mobile viewport model, a horizontally overflowing
document widens the INITIAL CONTAINING BLOCK — and the ICB is what
`position: fixed` resolves against.** Measured on a real phone
emulation: `ShelfScreen`, declared `position: fixed; inset: 0`,
computed to **728px** in a 390px viewport, putting its close button and
every row's actions off the right edge, **unreachable**. The command
sheet inherited the same fate. The surface was not merely ugly; it was
unusable, and no amount of `box-sizing` on the surface itself could fix
it, because the containing block was the thing that was wrong.

So `AppContainer` refuses to exceed `100vw` when compact, and
`ContentRow` takes the overflow into **its own** scroller. The rail
stays reachable by scrolling the *content* — where a phone user would
look for it — instead of by scrolling the whole page, which broke the
chrome. Clipping it instead would have made a shipped card unreachable,
which is a worse answer than the bug.

⚠⚠ **Neither suite could see this, and one of them lied by omission.**
jsdom has no layout at all. And Playwright's plain `viewport` is a
DESKTOP context that merely happens to be narrow — the ICB stays put,
every fixed surface measured a tidy 390px, and eight e2e specs passed
green over a broken build. The phone specs now pass `isMobile: true`,
and that flag is the difference between a test that models a phone and
one that models a small window. Verified by removing the fix and
watching the assertion fail with *"shelf-screen runs past the right
edge — expected ≤391, received 730"*.

The chrome's own claim is unchanged and still checked twice per
surface: it must not widen the page past the baseline it opened over,
and its own box must fit the viewport.

## The mobile bar

`components/frame/MobileFrame.tsx`, rendered by `App` in place of
`Frame` below the breakpoint.

```
row 1   [seal] [connection] [identity ▾]        ← never removable
row 2   [ GLANCE-LINE — head of the shelf ] [⌄] ← the grab
```

⭐ **The governing sentence: *a bar that wraps has nowhere to wrap
to*.** Desktop's shelf grows a second row when you pin too much and the
page absorbs it. On a phone every row the chrome takes is a row **the
feed loses**, and the feed is the app. So the shelf cannot live in the
bar at all.

`Seal`, `ConnectionChip` and `AccountMenu` are the **same components,
unchanged** — the fixed facts are not mobile copies, because a copy is
how two surfaces start disagreeing about one socket.

⚠⚠ **The bar OPENS the `self` subscription (`useSelfFigures`), and
forgetting to was this build's most embarrassing bug.** The figures
live in the store, but the subscription that fills them was a
`useEffect` inside `Shelf` — which the mobile bar does not render. So
the glance-line and the pull-down shipped reading a `shelfFigures` that
**nothing ever populated**: every row rendered its honest empty state
forever, on a phone, for figures the server was perfectly willing to
send. Eleven green tests missed it and *could not* have caught it —
every one seeds the store directly with `useStore.setState`, the seam
that makes the shelf testable without a socket, and is therefore blind
to *does anything ask?* **A derive-on-read surface needs its WAKE
tested, not only its read.** The subscription is now a shared hook and
`MobileFrame.test.tsx` asserts the bar opens it.

⭐ **The glance-line is `shelf.slice(0, GLANCE_ROWS)` — the HEAD of the
one list, not a second key.** The alternative was a `cockpit.glance`
naming the bar rows explicitly, and two lists that must agree is the
drift this codebase keeps warning about: an unpin that forgot the glance
key, a glance row that is not on the shelf, a `list` output that cannot
say which is true. One ordered list has one answer.

The consequence is that **choosing what rides the bar is *reordering***,
which is what [`cockpit shelf first`](./cockpit.md) exists for.
`GLANCE_ROWS = 3` is a client constant and has to be — the server does
not know how wide a bar is. A shorter shelf simply shows what it has; an
empty one says *nothing pinned* rather than leaving an unexplained gap.

⚠ The glance-line **scrolls** where the desktop shelf **wraps**, and
that is the one place the two disclosures legitimately differ: wrapping
is what a phone cannot afford, and the rows that do not fit are not
hidden behind a gesture — the pull-down shows the whole catalogue.

⚠ **`ViewsMenu` and Settings move into the account dropdown.** They
survive, because dropping them would regress two shipped surfaces, but
neither earns a permanent slot in a two-row bar when the identity chip
is already a dropdown holding account actions. `AccountMenu` takes an
`extras: ReactNode` slot rather than a `mobile` boolean — the component
should not know what a viewport is; whoever composes the bar does.
`ViewsMenu` gained an `inline` variant (list in flow, no trigger of its
own) so the **command strings are still built in exactly one place**; a
second component would be two sources for one menu entry. Its absolute
popover would also overflow a 320px screen leftward from a
right-anchored host, which is the `＋ widget` failure in the other
direction.

⚠⚠ **No notification bell**, asserted by name in both the unit and e2e
suites. The art puts `◔ 3` in row one and calls it never removable; it
is not built, not hatched and not placeholdered, because what belongs in
that tray is whatever the receiver *said* they wanted — which wants
`NotifyPolicy` read first — and nothing about a smaller screen changes
what is behind it. `SocialNotificationsPanel` stays reachable from the
account menu, so the capability has a home; it does not get a permanent
slot in the scarcest row on the screen.

### The pull-down, and the shelf screen

`ShelfPullDown.tsx` discloses the shelf **two-up** with the `card`
variant, **over** the feed rather than displacing it — pushing the
transcript down would reflow everything the player was reading, and
closing it would scroll them somewhere they did not ask to be. A phone
can afford height here and never width, which is exactly why `card` and
not `chip`.

`ShelfScreen.tsx` (reached from the pull-down) is the chooser: the
glance-line as its own named section — *three fit on the bar without
pulling down; everything else is one drag away* — above the full
catalogue with each row's state and reason. Every affordance sends a
`cockpit shelf …` command and **mutates nothing locally**; `to bar`
sends `first`, which pins an unpinned row at the head so *"put this on
my bar"* stays one tap.

⚠ Both **import** `SHELF_CATALOGUE`, `HATCH_COPY`, `figureFor` and
`pinnedShelf` from `Shelf.tsx`. Two copies of the hatch reasons is
precisely the decay the three-category table exists to prevent — the
second copy is where a reason gets softened into a generic *not wired*
by whoever is in a hurry. Likewise one `figureFor`: a live row added in
one place and not the other would show a number on one form factor and a
hatch on the other, for the same field, at the same instant.

⚠ Hatch reasons render as **visible text**, not a `title`. A phone has
no hover, so a tooltip reason is a reason nobody can read — which would
make *"each with its reason"* true of the markup and false of the
surface.

⚠ **Identity and connection are absent from the chooser by design.**
They are not shelf rows at all; `cockpit shelf unpin identity` refuses
with *unknown shelf row*, and that refusal is a stronger guarantee than
a rule in a component somebody could edit.

⚠⚠ **`box-sizing: border-box` on every full-bleed surface is
load-bearing.** `inset: 0` (or `left`/`right`) resolves the width from
the edges and the default content-box then adds padding *outside* it —
which made the shelf screen 18px wider than the viewport and scrolled
the whole page sideways. The client has no global border-box reset, so
each surface says it for itself. Found by driving; jsdom stayed green.

### ⭐⭐ The dropped row — what is true, not what is comforting

`DroppedRow.tsx` replaces the fixed facts entirely when
`link !== 'connected'`. On a phone you are usually not looking, so the
bar must not understate a drop — and the fixed facts are what a dead
link makes meaningless anyway. The glance-line below **stays**: a shelf
that vanished on a blip would reflow the bar every time a train went
into a tunnel.

The reference art's dropped panel is the most persuasive in the set and
the one that most overpromises — *"2 commands held · retry in 4 s"*, with
a footer naming the commands it kept. **No such queue exists.**
`WebSocketClient.send()` logs an error and drops the message. Building
one is an offline queue with ordering, expiry and replay-safety
questions — *is `north` still the right command forty seconds later?* —
which deserves its own requirements rather than being a chrome build's
side effect.

So the row reports:

| Reported | Source |
|---|---|
| reconnecting or dropped | `connection.link` |
| the retry countdown, live | `connection.retryAt` |
| a manual retry | the existing `reconnectNow` path, not a second implementation |
| how long since the last frame | `store.lastFrameAt` — see below |
| ⭐ **that commands sent now will not arrive** | the truth |

⭐ **That last row is the inversion worth making.** The art comforts —
*we kept your commands*. The truth is bleaker and strictly more
actionable: **they are dropped**. A player who knows their input is
going nowhere stops typing; a fabricated count of things allegedly saved
lets them keep typing into a void and believe it is being kept. It is
also the sentence that makes the queue's absence **visible** rather than
silently wrong — and when the queue is built, this is the row it
replaces. A test asserts no *held* count is rendered, by regex, because
the absence of a promise has to be guarded or it returns as a
placeholder.

⚠ The countdown is driven in test by advancing the clock rather than
asserted as a constant: a static *retry in 4s* that never moved would be
a worse lie than showing nothing, because it looks live.

⚠⚠ **`lastFrameAt` is a separate store slot precisely so it can outlive
the buffer.** The obvious source — `frames[frames.length - 1]` — is
structurally absent exactly when the row needs it, because
`WebSocketClient.onclose` calls `clearFrames()`. Driven on a real
server restart, the figure read *"no frame has arrived this session"* in
a session that had just rendered a full transcript. Scrollback is
session-scoped; *when the link last carried something* is a fact about
the LINK, and dropping it with the scrollback made the one surface that
needs it unable to answer. Its test now performs the same `clearFrames`
every real drop performs — the original seeded `frames` directly and
passed against the bug.

### ⭐⭐ Reporting a schedule obliges the schedule to be real

Driving a server restart — the standup-deploy path the generous
7-attempt window was written for — showed the client **never
reconnecting**: `reconnecting` forever, with the server back and healthy
for minutes, until the player reloaded by hand.

The race, from the console of a real drop:

1. `onclose` nulls the socket and arms attempt 1.
2. `setDisconnected` flips `connection.isConnected` false, which
   re-fires **`App`'s connect effect** — a second, independent caller of
   `connect()`, out of band with the backoff.
3. That immediate attempt fails and arms attempt 2.
4. Attempt 1's timer fires and opens a socket.
5. Attempt 2's timer fires, finds it mid-handshake, and returns
   **without re-arming**. The chain is lost.

`WebSocketClient.reconnectTimer` makes the backoff loop the **sole
driver**: `connect()` refuses while an attempt is armed, so step 2
cannot inject a competing socket, and the loop stays one ordered chain —
close → arm → connect → close → arm. `reconnectNow` and `disconnect`
disarm first, since a pending attempt must never block the manual
affordance or resurrect a session the caller just ended.

⚠ **This is a behaviour change to a machine this build's requirements
fenced off as additive-only, and it is deliberate.** The fault is
pre-existing; what is new is that the retry countdown made it legible.
A countdown is a promise, and reporting when the next attempt fires
obliges the next attempt to actually fire. The three frozen guard files
still pass unmodified.

## ⭐⭐ One strip, and every tab is a VIEW over the whole buffer

`All 252 · Aether 0 · Diag 87 · Forge watch 109 ⋯ × · +`

A tab is a **named predicate**, evaluated at render over every frame in
the buffer. Shipped views (`FILTER_PRESETS`) and the player's own sit in
one list, interleaved, because they are the same kind of thing — which
of them happens to ship with the client is not a distinction a reader
has to care about.

⚠ Counts are derived from the predicate, over the **unfiltered** buffer.
The count and the contents are the same rule applied to the same frames,
which is also what makes a view retroactive: changing one re-sorts your
whole history rather than affecting only what arrives next.

### ⚠⚠ Routed feeds were retired, and why

There used to be a second strip above this one —
`World | Attention | Channels | Diag` — where the routing table MOVEd or
COPYed each frame into one destination at delivery. It was the same act
performed by a second control, and two properties finished it, neither
patchable inside the destination model:

- **The stamp was applied at delivery**, so changing a rule never
  re-sorted history; old frames kept their old routing forever.
- **The frame store does not persist the stamp**, so on reconnect every
  backfilled frame fell to `world`. Measured live:
  `World 244 · Attention 1 · Channels 0` — the non-World counts were
  only what had arrived since the socket opened. The player's dm history
  was not in Attention.

A predicate has neither problem. **MOVE/COPY disappears with the
buckets**: a frame is simply in every view whose predicate it satisfies,
which is what COPY was straining to fake.

⚠ The rules themselves are retained — `DEFAULT_ROUTING`, the predicate
vocabulary, `MessageApi.feedsFor` and its tests — as the substrate for
**notification policy**. *Which frames should ping you* is a real
question that wants exactly this shape. Its UI returns when there is
something for it to drive; a settings screen for rules that change
nothing observable would be a control lying about its own effect.

### ⚠⚠ `All` is locked, and `Aether` is the only default

Two different things sit in the strip, and the difference is *what they
are*, not a privilege list:

- **`All` is the ABSENCE of a filter.** It is not stored in
  `console.tabs` at all — it is a structural first entry. There is
  nothing in an empty predicate to edit and nothing to delete, so it
  carries no `⋯` and no `×`. It is also the floor: whatever the player
  removes, the unfiltered view is always there, so they can never end up
  with nowhere to look and no re-seeding is needed.
- **Everything else is the player's.** `DEFAULT_VIEWS` seeds exactly one
  — `Aether` — and the moment it lands in `console.tabs` it is ordinary
  data: select, edit, rename, delete, exactly like one composed by hand.

⚠ The lock holds against **state that predates it**. A player seeded
before `All` was structural still carries a stored `All` row, possibly
with facets on it; the strip renders it once (never twice) and `App`
resolves the active view to `undefined` for `All` rather than looking it
up, so a stale edit cannot make the locked view quietly filter.

It shipped wrong twice on the way here, and both are worth keeping
written down. First, three shipped views that could not be touched sat
in the same strip as one that could — and selecting a shipped one
re-applied its filter **from code**, so tuning `Aether` and clicking
away lost the change silently: *"'forge watch' I can edit but aether and
diag I can't? I thought a filter was a filter."* Then the correction
over-swung and made `All` deletable like the rest, which mistook the
identity for a member.

- **Seeding is additive and one-time.** `console.seededViews` records
  what has been OFFERED, not what exists, so a deleted view stays
  deleted instead of returning on the next login. Without it "delete"
  would mean "hide until you reconnect". ⚠ The key must be declared in
  `HasInteractiveMixin.clientStateSchema` — it shipped missing, so every
  write of it was rejected and logged, and the guarantee was quietly
  not there.
- ⚠⚠ **Absent client state means NOT LOADED YET, and nothing is
  written.** The seeder read an absent `console.tabs` as "first run" and
  wrote the ship defaults wholesale — a write of defaults over state it
  had not read. `App` renders the layout on its `default:` branch, so it
  can mount before the connection payload lands, and the seeder then
  clobbered the player's saved views. Found by driving: two views
  composed on a desktop were simply gone on the next connection, and the
  persisted document confirmed they had never been written. The seeding
  effect is keyed on the value being present, so it re-runs when state
  arrives and seeds additively then.
- **No preset row inside the editor.** Chips named after the seeded
  views used to sit at the top of the facet panel and overwrite the
  filter being edited — inside an editor titled *Filter — Forge watch*
  that is a third mechanism, neither the view you are editing nor the
  strip you chose it from.

### Composing one

`+` **names it, activates it and opens the editor in one gesture.** The
earlier version made an empty tab and stopped, leaving the player to
work out that they had to select it and then find a separate gear before
it meant anything — *"it's not obvious that I have to go back after
hitting + to set my filters."* A control whose effect you can only
discover by leaving it and coming back is not a control.

Edit (`⋯`) and delete (`×`) ride the **active** view only: a row of
controls on every tab is noise, and on the one you are looking at they
are where you reach for them. The editor carries the three facet axes
with live per-value counts, a `SHOWING n of m` readout, and the
topic-mute tree beneath it.

⚠ There is exactly **one** entry to that editor. A second in Settings
would give no way to tell, from that screen, which view it was about to
change.

### ⚠⚠ `Aether` is a topic list, and that is a finding

Facets are the default, for the reason they were built: one rule does
not drift when a topic is added. But **`speech.vocal` and
`speech.channel` carry identical facets** — both `address: broadcast,
actor: person` — so no combination of the three axes separates *talking
out loud in a room* from *talking on a channel*. A view meaning "the
electronic layer" cannot be a facet rule.

`FacetFilter.topics` is therefore a topic-prefix **allowlist** — the
include direction of the tree half the model already had, since
`ConsoleTab.muted` is the same operation pointed the other way.

⚠ Worth a separate look: two topics a player experiences as completely
different things being facet-identical is arguably a gap in the S2
taxonomy rather than in the filter. Fixing it there would let `Aether`
go back to being one rule.

## The phone's play surface

The rule is **interleave what is causally related, switch what is
independent**:

- **Cards render inline in the feed**, in causal position. You analysed
  the forge; the forge's card belongs where that happened — not in a
  second column, and not in a drawer you forget exists.
- **Named views get the strip**, because there is no causal reason a
  channel message should sit between two room descriptions — but which
  messages you are reading is a predicate you choose, not a place a
  rule sent something.

Getting this backwards produces the two familiar mobile failures: a
drawer you forget exists, and a single stream where unrelated things
fight for the same position.

- **A pinned chip row** sits above the command bar — pinned cards are
  the ones you told the world to keep, so on a phone they keep a
  permanent handle instead of scrolling away.

⚠ **The left-behind card is gone**, with the routed feeds it belonged
to. A stub reading *"one `speech.channel` went to Channels — open
Channels"* only means something when feeds are exclusive destinations. A
frame is now in every view whose predicate it satisfies, so nothing is
ever routed out of view and there is nothing to leave behind.

⚠ **Copy-to-Attention is gone for the same reason.** It shipped on by
default on a phone as the safety net for "World may not be the feed you
are looking at" — a problem that only exists when a MOVE can take a
frame somewhere you are not.

### ⚠ Screen budget is a design constraint, not a detail

Found by driving at 390px: the routing toggle and the prompt format bar
each took a full row above the input, and with the format bar's four
token chips wrapping onto three lines they held **~180px of an 844px
screen permanently** — over a fifth of the phone, for two controls that
edit settings rather than play. They share one 52px row now, and the
tokens collapse behind a single control.

⚠ Re-check the **ICB trap** at 390px on every pass: a fixed-width card
inside an overflowing document widens the initial containing block and
pushes `position: fixed` chrome off-screen. jsdom lays out nothing, so
only a real browser with `isMobile: true` can see it.

## The widget shelf

`components/frame/Shelf.tsx`. The player-pinned row of figures across
the top bar, and **the first consumer of the honest-state primitives**.

⭐⭐ **Three of the nine rows have a live server read and six do not.**
A shelf showing nine confident numbers would be lying about six of them,
so the six say what they are instead.

⭐ **But the default shelf pins only the three that are wired.** An
earlier cut defaulted to all nine, reasoning that the shelf being mostly
hatched *is* the convention working and should be visible on first
login. That mistook a principle for a product: a new player's first
impression should not be six dead boxes, however truthfully each one
explains itself. **Never default-pin a widget that does not do anything
yet.**

The convention is not lost — it moves to where it is useful. The
`＋ widget` menu carries the whole catalogue with every reason in
visible text, which is exactly the moment a player is asking *why can't
I have COIN?* A reason shown at the point of the question beats a reason
shown on a bar nobody asked about. So the **nine is the catalogue**; the
**three is the default**, and a row joins the default when it starts
answering.

| Row | State | Default-pinned | Category |
|---|---|---|---|
| `PLAY` | **live** | ✅ | `playStanding` |
| `RENOWN` | **live** | ✅ | `renown` |
| `SKILL` | **live** | ✅ | `practisingCompetence` |
| `MAKE` | **live** | — | `makeStanding` |
| `COIN` | hatched | — | `unexposed` |
| `STATUS` | hatched | — | `unexposed` |
| `TIME` | hatched | — | `not-self` |
| `ONLINE` | hatched | — | `not-self` |
| `DOCKET` | hatched | — | `not-self` |
| ~~`TRAIT`~~ | **never** | — | see below |

### ⭐ The hatch categories, and why they are distinct

The reasons are distinct claims and must not collapse into one. They
tell the next builder **where to look**, and a single generic "not
wired" erases exactly that:

1. ~~**`level` — the account gap**~~ (`MAKE`) — **RETIRED in the Arrival
   build.** It said the value existed but its *level* was wrong:
   `Avatar.makeStanding` returned a real band while *Make* is an
   account-level stock (`STOCK_LEVEL`) whose arithmetic was unbuilt, so
   the number answered per-character for a per-person claim. **A figure
   whose level is wrong is not a figure you can render** — the honesty
   rule applies to the level, not only to the value. The account
   roll-up landed ([influence.md](./influence.md) §
   *`standingForHost`*), `MAKE` went live, and the category went with
   it. ⚠ It was found stale by *driving the client*, not by the suite:
   the row still printed "the account arithmetic is unbuilt" after the
   arithmetic shipped. **A hatch reason is a claim about the code and
   goes false silently** — nothing type-checks it.
2. **`unexposed` — the missing field** (`COIN`, `STATUS`). Genuinely
   figures about you, simply not on the wire. A one-field addition
   unhatches each.
3. ⭐ **`not-self` — the wrong scope** (`TIME`, `ONLINE`, `DOCKET`).
   These are **world** figures. The `self` card structurally cannot
   carry them *no matter what is added to `Avatar`*; they need a
   different source — a world-scoped card, or a different subscription
   entirely — and that is a design conversation, not a field addition.
   Recording the distinction is the point: a reason saying "no
   subscribable field yet" would send the next builder into `Avatar`
   looking for something that can never be there.

The reason is derived from a **category**, not typed per row — the
`contrast.test.ts` totality-gate pattern. A row must be *classified*,
the categories are strings in one table, and a test asserts they are
pairwise distinct — which is also what made retiring `level` a
one-table edit rather than a hunt. A per-row free-text reason decays into the
generic one the first time somebody copies a neighbouring line.

### ⚠⚠ `TRAIT` is permanent, not deferred

The reference art lists it. The psychology vocation rests on self-other
asymmetry, and a pinned always-on readout of your own personality is the
stat sheet that makes the therapist unnecessary. `ShelfRowId` omits it,
the server's guard forbidding subscribable field names matching
`trait|disposition|personality` stands unmodified, and `Shelf.test.tsx`
asserts its absence by name — the client-side twin of the same wire.

### What is deliberately NOT on the shelf

`LoadBearingMixin` contributes three MORE live self fields —
`borneBurden`, `carryCapacity`, `loadRatio` — that the catalogue does
not list. The catalogue is the reference art's, and widening it is a
design decision. **The shelf's live set is limited by the catalogue,
not by the wire.**

### The `＋ widget` menu, and pinning

Clicking a catalogue entry sends `cockpit shelf pin <row>` /
`unpin <row>` and mutates nothing locally; the server writes
`cockpit.shelf` and pushes it back. Hovering previews the identical
string in the status bar — the same constant, so preview and send cannot
drift into two call sites that merely agree today.

⚠⚠ **A row's state column names its ACTION — `pinned` or `add` — never
a placeholder.** It read `pinned` or `—` until the Wave 1 mock audit,
and `—` is the glyph an *empty figure* uses, so an unpinned row wore a
broken row's costume. `MAKE` made it concrete: it went live with the
account roll-up, and a live row carries no reason line, so it sat under
a bare `—` reading as **more** broken than the hatched rows around it,
every one of which at least explained itself. Every item here is a
button that does something — labelling one with the vocabulary of
absence describes the *figure* where the column describes the
*affordance*. The phone's shelf screen always named its actions (`pin`
/ `to bar` / `remove`); this was the desktop half catching up.

The 30px chip has no room for a visible reason line, so the reason rides
`title` **and** the `aria-label` `Figure` emits — and they are the
**same string**, built once (*"none — &lt;reason&gt;"* for an empty
figure, *"not wired — &lt;reason&gt;"* for an unwired one, in words, so
a screen reader gets it). ⚠ The `title` was once built separately with
`not wired` hardcoded; see § *The honest-state primitives* for why that
made an `empty` chip lie. The `＋ widget` menu is where every row's
reason is **visible text** — the chip is the compact face of the same
fact, never the only place it appears.

⚠ The menu carries more weight than it looks like it should, because
the hatched six **do not start on the bar**: it is the only place a
player meets them, so it is where the honesty convention does its work.
It is anchored to the RIGHT of its button — left-anchored it overflowed
the viewport and made the whole page scroll sideways, which only a real
browser could see.

The shelf takes `flex: 1; flex-wrap: wrap` — **wraps, never scrolls**.
Nothing pinned may be out of sight; a horizontally scrolling shelf hides
figures behind a gesture, which is the same failure as not rendering
them. Driven at 900px in `e2e/tests/shelf.spec.ts`.

### The subscription

One `self` card, opened by name in a `useEffect` and torn down on
unmount — the shape `InspectionCard` established, with no new service
layer and no second registry. Results land in the store
(`shelfFigures`) rather than component state, so tests drive the shelf
without a socket.

⚠ **The single-cardinality delta trap applies.** `self` is
`cardinality: 'one'`, so a slot replacement arrives as one `replace`
keyed by the NEW stuffId and the generic `applyChanges` would append a
second record; an `update` carries only what changed and must MERGE.
The location handler's bypass is followed. See
[mql-subscription.md](./mql-subscription.md).

## The status bar

`components/frame/StatusBar.tsx` — the global footer, and the surface
that makes *the command line is never silent* true rather than
aspirational. `CONVENTIONS.md` #5, and § 3.5's axiom: **every click
sends a command, and the interface shows which.**

It replaced `GhostCommandLine`, which is deleted. The relocation was
smaller than it sounds — the ghost line already sat at the bottom of
`AppContainer`, below the whole content row, so it was already a global
footer. What changed is its shape (a `flex:1` ellipsizing preview
region, a `flex:none` right region), its at-rest copy, and its home in
`frame/` beside the other chrome.

⚠ **Exactly one preview surface, guarded structurally.** Two places
showing what a click would send is worse than none, because they can
disagree — and a disagreement here does not merely look wrong, it
discredits the one claim the surface exists to make. So
`StatusBar.test.tsx` scans `src/` and asserts exactly one module reads
`ghostPreview` and nothing imports the retired component. A second
consumer fails the suite the moment it is written.

Two RENDER sites remain and both are correct: `App` (in-world) and
`CharGenStage`. They are mutually exclusive phases — `App()` is a switch
with an early return per phase — so exactly one bar is mounted at any
instant. **Char-gen keeps a preview surface and must**: it renders
affordances that send commands, and the axiom does not switch off during
intake.

⚠ **The in-world bar is absent below the breakpoint**, not hidden. It
has no hover to report on a phone and would cost the feed a row. What
replaces it is the command sheet.

## ⭐ The command sheet — a tap names its command before it sends

`components/frame/CommandSheet.tsx`, the phone's replacement for hover.

Desktop hover asks *what would this send?* — there is a moment before
you commit, and the status bar fills it. **On a phone there is no
before: a tap is the commit.** So the sheet is not a preview surface
with a different shape; it is the moment a phone otherwise does not
have, inserted deliberately. Tapping any affordance opens a sheet naming
the **verbatim** command; the player sends it or does not.

§ 3.5's axiom is preserved and arguably strengthened: the interface
still shows which command, and still *before* it goes. What changes is
that the showing costs a tap instead of a hover.

⭐⭐ **The interception is at `App.handleCommandClick`, and that
placement is the whole design.** Every affordance in the tree —
transcript tags, shelf entries, `ViewsMenu`, the pull-down, the
right-column cards — routes through that one handler, so the confirm
step is **one interception point for the entire app** rather than an
`isCompact` prop threaded into every renderer. `MmlRenderer`,
`EntityName` and `Shelf` needed no changes at all. One `sendDirect`
serves both the desktop click and the sheet's confirm, so *what the
sheet showed* and *what got sent* cannot become two strings that merely
agree today.

⚠⚠ **"Every affordance" is a claim about WIRING, and it was false for
one family until it was driven.** The four right-column cards
(`InspectionCard`, `WhoCard`, `NewsTickerCard`, `WikiCard`) took
`onSendCommand` — the raw send — so they bypassed the sheet entirely:
tapping `north` in the transcript opened a sheet naming the command,
while tapping the identical `north` in the card six inches away sent it
instantly. **Two rules on one screen is worse than either rule alone**,
and it is precisely the unpredictability the no-exceptions policy above
exists to prevent. `WorldLayout` now passes `onCommandClick` to all
four, and `layouts/__tests__/affordanceRouting.test.ts` guards it —
because a new card wired to the wrong prop is one copy-paste away.

⚠ `CommandBar` keeps the raw send, correctly: **typed input is not an
affordance** and must never be confirmed.

⚠ Every unit and e2e assertion about the sheet happened to pick a
transcript or menu affordance, so a fully green suite said nothing
about this.

⚠ **Every affordance, with no "obvious case" exception.** An unambiguous
`look anvil` gets a sheet too. The extra tap **is** the pedagogical
dividend — clicks are how a player learns the command line exists — and
a rule with an exception for cases the client judges obvious is a rule
nobody can predict, which is worse than either uniform answer.

⚠⚠ **It does not read `ghostPreview`.** The sheet carries its own store
slice (`commandSheet: string | null`). The one-preview-surface guard
above passes **unmodified**, and a second assertion states the same
claim from the sheet's side — because *"what a hover would send"* and
*"what this tap will send, pending confirmation"* really are different
facts. A sheet that reused the preview slice would trip the guard *and*
be conflating two concepts; the guard catching it would be correct
behaviour, not a false positive.

⚠ The sheet shows **one** command, because that is what an affordance
affords today — `commandFor(node)` returns a single string. It renders
in the command register (mono), verbatim and wrapping rather than
truncating: an ellipsis would hide the half of the command a player most
needs to see before sending it.

⭐ **The art's at-rest right region reads `here:forge · 1,240 frames`,
and neither figure is rendered.** Nothing measures a frame count, and
`here:` would be a location readout with no subscription behind it. The
right region carries `click to send` while previewing and nothing at
rest. Painting two invented numbers in the surface that advertises that
the interface tells the truth would be the joke this build is not
making.

## The connection popover

`ConnectionChip`'s expansion — three readings, of which two now have a
source:

| Row | State | Why |
|---|---|---|
| this connection | **live** | derivable from a connect timestamp; no server work, no new wire field |
| round trip | **live** | measured by the service-owned heartbeat — see below |
| frames behind | hatched | needs a server sequence number, which genuinely does not exist |

The popover's own copy is what earns it a place — *a dropped socket in a
MUD costs you whatever you were mid-way through* — and the honest
version of that surface says which of its readings it can stand behind.

### ⭐⭐ Round trip, and a hatch reason that was WRONG

Round trip shipped hatched, with a reason saying nothing measured it and
a ping/pong would have to be written. **That reason was inaccurate**,
and the correction is the most instructive thing in this section.

The protocol existed end to end the whole time.
`websocketClient.sendPing()` sent a client timestamp and
`backend/inbound/ping.ts` replied. What was missing was that nothing
*called* one and nothing *handled* the other — the `pong` carries no
`frameId` and no `topic`, so it fell through the envelope discriminator,
reached the catch-all as a topic-less frame, and was dropped in silence.

⚠ **A reason that points at the wrong place is worse than no reason at
all.** The whole purpose of classifying a hatch is to tell the next
builder where to look; this one would have spent somebody's afternoon
writing a second copy of a protocol that was already there. That is
precisely the decay the three-category hatch table exists to prevent,
and it happened anyway — which is why the retired string is now guarded
by a test (`roundTripReason.test.ts`) that greps the client source and
fails if it survives anywhere, including in prose. `ConnectionChip`'s
own comment paraphrases the old reason rather than quoting it, for
exactly that reason.

What the fix consists of:

- **A service-owned heartbeat.** `WebSocketClient` starts a **30s**
  interval on `connection-established` and clears it on close, error and
  the manual reconnect (which detaches `onclose`, so it must stop the
  timer explicitly). One ping fires immediately so the figure is not
  blank for the first half-minute. ⭐ It lives in the service and not in
  the popover because a component-owned ping would only measure while
  somebody was looking: the number you read on opening would be the
  first sample rather than the current state of a socket that has been
  up for an hour. The reading is true whether or not anyone has asked
  for it.

  ⚠ Round trip is a **popover** row on both form factors — the mobile
  bar reaches it through the same `ConnectionChip`, not as a chip at
  rest. An earlier draft of this section and of `websocket.ts` claimed
  the bar showed it at rest; it never did, and the claim was corrected
  in the pre-merge sweep. The service-ownership argument above stands
  on its own and never depended on it.
- **The server echoes the client's own stamp.** `pong` now carries both
  clocks: `timestamp` (the server's, unchanged) and `clientTimestamp`
  (echoed verbatim). ⚠ Subtracting the *server's* timestamp measures
  clock skew between two machines plus half a trip — a number that looks
  like latency and is not. Echoing makes each pong **self-describing**,
  so the pairing rides the protocol instead of a private slot that
  cannot tell whether the reply it got belongs to the ping it sent. The
  stamp is echoed, never trusted: nothing server-side reads it.
- **`ConnectionState.roundTripMs`**, optional, absent until the first
  pong lands and cleared on a drop — a stale trip measured on a dead
  socket would sit as a confident latency reading beside the word
  "dropped". A negative sample (the clock stepped under us) is discarded
  rather than printed.

⚠ Frames-behind stays hatched and is now the only unmeasured claim here.
Its reason names something that genuinely does not exist.

⭐ **The duration row says "this connection", not "session".** A
successful reconnect issues a fresh `connection-established`, so
`ConnectionState.connectedAt` resets. Rather than paper over that with a
fake continuous session clock, the label names what is actually
measured. Same move as hatching `MAKE`: the honest fix for a figure at
the wrong level is to correct the *claim*, not the number. With no
`connectedAt` at all the row is `empty` with a reason — never a
fabricated `0m`.

`connectedAt` is an **optional** field on `ConnectionState`, and the
optionality is load-bearing twice. Practically: `connectionLink.test.ts`
builds a complete `ConnectionState` literal and the client's `tsc`
includes its tests, so a required field would break a frozen reconnect
test under `build:types` while still passing `vitest` — the worst
failure shape there is. Honestly: before the first connection there is
no timestamp, and a sentinel would be a value standing in for an
absence.

The ticking is a `useEffect` interval **inside the popover, only while
open** — never a global 1 Hz re-render of the bar.

⚠ **The reconnect machine's BEHAVIOUR is untouched.** The link
vocabulary, the backoff schedule and `ReconnectBanner` are unchanged,
and the three guard files (`connectionLink.test.ts`,
`ConnectionIndicator.test.tsx`, `websocket.test.ts`) plus
`ConnectionIndicator.tsx` still show **zero diffs**. If one of them ever
needs changing, that is the signal that behaviour moved where only
reporting should have.

### ⭐ The retry countdown — exposed, not invented

`attemptReconnect()` has always computed an exponential backoff
(1·2·4·8·16·30·30s) and then **dropped it into a bare `setTimeout`**. The
store learned only `link: 'reconnecting'`, so every surface could say
*we are retrying* and none could say *when*. Same shape as the renown
measurement: the information exists and is thrown away one line before
it leaves. Surfacing it is not new measurement, and the schedule itself
did not change — the same number is now told to somebody.

`ConnectionState.retryAt` is an epoch-ms stamp, optional, present only
while an attempt is actually scheduled. ⚠ Once the machine gives up
there is no *next* attempt, so the field is absent and the countdown
reads `empty` with a reason — a clock still ticking against a link that
will never retry on its own would be the most misleading figure on the
screen.

It rides `setDisconnected` as an **optional third parameter** rather
than a second store action. Optional keeps every existing call site,
including the frozen test's 0-, 1- and 2-arg calls, compiling and
behaving identically. One action rather than two keeps the transition
atomic: a caller that updated the link and forgot the countdown would
leave a stale clock ticking against a connection that is already back.
`setConnected` clears both derived readings **by construction** — it
replaces the connection object rather than merging, so neither can
survive a reconnect through somebody forgetting to reset it.

## ⚠ The read-only mode indicator: cut, not deferred

The client-slate's Build B line named a read-only mode indicator. It was
**cut during planning because it has no source**, and this is recorded
so the line is not later read as an unmet promise.

The only read-only principal in the system is the livestream broadcast
feed, and [livestream.md](./livestream.md) records that the broadcast
connection *"has no `Interactive` at all"* — it is an out-of-band socket
for overlays, registered straight with `BroadcastFeed` as a pure push
target and absent from the connection registry. It never receives
`connection-established`, so no flag on that payload could reach it, and
`packages/client` has no reference to the feed at all. Nor is there a
read-only `CockpitMode`: `watch` is a mode, but a watching player holds
a full `Interactive` and can act.

So the indicator would have had nothing to indicate. Building one would
have meant inventing a read-only session state to justify a chip — the
interface leading the model, which is how a surface ends up asserting
something the server cannot back.

⚠ If a read-only React session is genuinely wanted — a spectator link, a
shared-screen mode, a suspended account that can read but not act — that
is a real feature with server work and wants its own requirements.

## The front door

`components/StartScreen.tsx` renders the `unauthenticated` phase — no
diegetic metaphor (the lounge is the first room; this is the app's front
door). It carries the pitch, the three chambers (Make / Fund / Play), a
**data-shaped provider list** (Google, Twitch and Kick as co-equal login
providers, each disabled when `/auth/status` reports it unconfigured),
the **guest** control with its terms, the {@link PressRoom}, and a
**dev-only** no-OAuth login (`import.meta.env.DEV`). Logout returns here
(a real logged-out state, never a reload to a dead page).

The composition is **full-bleed horizontal bands** — a hero band
carrying the seal, wordmark, headline and the sign-in column together,
then a chambers band, then the press room and the facts row beneath it.
The bands run edge to edge; only their contents are width-capped
(`BandInner`), so the page reads as a civic document rather than a
centred card on a page. Sign-in is a fixed 350px column inside the hero
band, and every provider is the same seal red with a white monochrome
mark — **Google gets no visual precedence**, because giving one
identity provider a distinct treatment states a partnership that does
not exist.

Compact is a **reflow**, not a different composition — the bands stack
to one column, sign-in above the fold, the press room as the scroll —
so it is plain CSS rather than a `useIsCompact` branch. The mobile bar
is the case that genuinely needs the hook; this is not.

⚠ The app ships a document reset (`styles/GlobalReset.ts`), mounted
**outside** `React.StrictMode` for the same styled-components reason
`GlobalFonts` is. Without it `body` kept the user-agent `margin: 8px`,
which put a white frame around every screen and made the document
`100vh + 16px` — a scrollbar on a page that fits. ⭐ The overflow read
as *content too tall* and the instinct was to compress the layout;
compressing would have made the design worse and left the scrollbar,
because the 16px was never in the content. Measure the container before
you shrink the contents.

⭐⭐ **Every claim on this screen has a source**, and this is the screen
where that is hardest to hold: it is what a stranger judges the project
on, so the pull toward filling space with plausible copy is at its
maximum. Two claims the handoff art shipped as fixed strings are now
server-reported:

| Art copy | What ships |
|---|---|
| *"it is usually quiet · you may be the only person on"* | a **live count** from `/auth/status`'s `online`. Three states, and the third is the point: a number, a real zero, or **nothing at all** when the server did not answer — an unanswered count must not render as "nobody is on", which is a different claim and a discouraging one to get wrong. |
| *"the world resets nightly · nothing survives to tomorrow yet"* | rendered **only** when `/auth/status` reports a `resetPolicy`, which is absent by default. |

⚠⚠ The second is not fussiness. The claim was **implemented nowhere** —
no cron, no CI job, no script — while three design documents reasoned
*from* it. A statement about whether the player's own work survives the
night is the most expensive kind to get wrong, and the fact that it is
*reassuring* is what makes it dangerous. There is no fallback string in
the component and there must never be one; `App.test.tsx` and
`arrival.spec.ts` both guard it.

The press room is **never awaited**: the sign-in and guest controls
paint whether it loads, is empty, or is gone.

## Character select

`components/CharacterSelect.tsx` renders the `character-select` phase.

⭐ **Every figure rides the roster payload**, because at `Login` you are
not embodied: the per-character standings are readable in-session
through a subscription and none of those is available to a screen with
no character. `lastSeen`, `playStanding`, `lastLocation` and `practice`
shipped on that payload in an earlier build and nothing read them until
this one. The **account** block (`account.make`) is the second entry
point to the standing roll-up — see
[influence.md](./influence.md) § *`standingForHost`*.

⭐ **Two cards on a desktop, two screens on a phone, and the split is by
QUESTION**: the list answers *who*, the detail answers *what happened
while I was gone*. Most accounts have one character, so the list is a
way-station rather than a destination — **a single-character account
opens straight on the detail** on both form factors.

⚠ Three hatches, three **distinct** reasons. Collapsing them into one
string would send the next reader to the wrong place, which is the
failure the honest-state convention exists to prevent and which an
earlier build shipped anyway:

| Surface | State | Reason |
|---|---|---|
| **Since you left** | whole-card `UnbuiltGround` | nothing records what happened while you were away — no mailbox, no notice queue |
| **Fund** standing | `unwired` | the capital stock has no faucet |
| **Make** standing, unresolvable account | `unwired` | the account could not be resolved |
| **Retire** | disabled, reason in `title` | no retire command exists |
| Play standing / practice / location, never-played | `empty` | *never taken out* |

⚠ *Since you left* is a whole-card hatch rather than a hatched value
because hatching one row inside an otherwise confident panel would
understate how absent it is.

⚠ A never-played character shows a reasoned `empty`, **never a zero**. A
zero says *we measured, and it is nothing*; the truth is that nothing
has been measured.

⭐ The Enter control previews the command it sends (`sends as play
<id>`) — the axiom does not switch off before the command bar exists.
It **leads the detail card** rather than closing it: this screen exists
to be left, and the one control that leaves it should not be the last
thing found. The screen's header band names the **account** — the
person, not any character on it — and carries the account's own meters.

⭐ **Retire is kept as a disabled stub; rename and appearance are
dropped.** The difference is whether the absence is worth advertising:
retiring a character is a thing a player will look for and not find, so
the disabled control with its reason is the honest answer. Rename and
appearance answer nothing anyone asked at this screen, and a disabled
control for an unasked question is clutter pretending to be candour.

## Intake on a phone, and the in-world rail

Intake is one column at phone width; the species detail card stops being
a fixed 260px card.

⚠⚠ **The in-world rail collapses on compact**, and it is in this build
rather than the play-surface wave for one reason: **the arrival path
terminates there.** Each right-column card declares its own fixed 360px
width, so beside a terminal the in-world document computed ~698px at a
390px viewport — and under the mobile viewport model an overflowing
document widens the **initial containing block**, which is what
`position: fixed` resolves against, so the shell's own fixed surfaces
(the shelf screen, the command sheet) rendered off the right edge. An
arrival that delivers you somewhere broken has not arrived.

⚠ **The collapse and nothing more.** What the cards should be on a
phone, and whether they belong inline in the feed, is the play-surface
wave's, and this must not pre-empt it.

## Anonymous guest

## Anonymous guest

A guest is **anonymous** (button on the logged-out screen) and **wholly
ephemeral**. Two orthogonal axes, deliberately not conflated:

- **Authentication (session/account axis)** — `POST /auth/guest`
  (`services/auth/GuestAuthRoutes.ts`) establishes an *anonymous*
  principal `anon:<nanoid>` (`User.ANONYMOUS_PREFIX`) via `req.login`.
  The WS upgrade gate is **unchanged** — it sees a valid
  `passport.user.id`. `Application.handleUserConnect` recognizes the
  prefix and builds an ephemeral, never-saved `User` (`user.anonymous =
  true`) instead of a Mongo lookup. The single admission gate is
  `GuestAuthRoutes.mayMintGuest` — flip it to demote guests to
  post-sign-in if abused.
- **Guest-ness (character axis)** — `Avatar.isGuest`. `Login.enter` is
  the **one** place the policy "anonymous session → mint a guest avatar"
  lives; the build itself is `Login.mintRandomGuestAvatar` (private),
  which reads the char-gen rosters + sex-set rule from `EnrollController`
  (`loadConfig` / `validSexSet`) so the guest and char-gen paths agree.
  Every pick is **randomized** — a random species, a random
  **non-intersex** sex, a random aspiration (→ bio + themed outfit) —
  pronouns are always **they/them**, the name is the reserved-word guest
  name, and the avatar spawns in the lounge (the seed's `startLocation`).
  Every guest *behavior* keys off the avatar's `isGuest`, never the
  session.

Guest lifecycle:

- **Minted on Enter** (not page-load). The build forks a **unique
  per-guest transient template** (`/obj/Avatar/guest-<nanoid>`) from the
  seed, overlaying the random picks, clones it, then **deletes the
  template** immediately — guests persist nothing, and the live avatar is
  independent of it. The unique path also means no two guests ever clone
  the same path, so there's no seed-clone concurrency hazard (no
  serialization needed).
- **Recognizable, non-impersonable.** The name is the reserved word
  (`Login.GUEST_RESERVED_WORD = 'Guest'`) + a NameBank-drawn
  distinguisher ("Guest Mallow") drawn only from the real `common`
  `NameBank` (an unseeded bank degrades to a bare "Guest", never a
  fabricated surname) — so guest-ness rides every attributed line
  (speech/emote/look), where a UI badge can't reach. The reserved word is
  on the char-gen `enroll` name denylist
  (`EnrollController.isReservedName`, which imports it from `Login`) so a
  real player can't impersonate a guest. Exact-word only; fuzzy/homoglyph
  is out of scope.
- **Persists nothing.** `Avatar.save()` short-circuits for guests (the
  single guard covering autosave, onDestruct, and the client-state
  don't-flush seam in `backend/inbound/clientState.ts`); autosave never
  starts.
- **Reaped on disconnect.** `Avatar.onLinkdead` destructs a guest
  immediately (no reconnect window — there's nothing to resume). The
  client routes a dropped guest to the start screen.

## Portrait — one method on the connection layer

Portrait resolution lives on **`HasInteractiveMixin.getPortraitUrl()`** —
the connection layer, present on both pre-world `Login` and in-world
`Avatar` — so the account photo is available from the moment of
connection. It is **one method**, not an Api (no `DescribeApi`
addition — that Api is being end-of-lifed; presentation resolves on the
host):

1. the `identity.portrait` **setting** (declared on `PersonaMixin`;
   `resolveSetting` returns `''` on an Avatar with it unset, `undefined`
   on a Login — both fall through), else
2. the connected account's Google photo (`Interactive.getUser()` →
   `googleProfileId` → `GoogleProfile.photoUrl`), else
3. `''` — the empty sentinel; the client renders a generated initials
   placeholder.

The account photo is **never written into the setting** (that would
freeze it stale); resolution is read-time only. The resolved URL rides
`ConnectionEstablishedPayload.player.portraitUrl`.

## Connection loss (linkdead) on the client

The spine: the server does **not** reap a real avatar on linkdead
(`Avatar.onLinkdead` just emits an event), so a real user can reconnect
any time and resume — be generous, preserve context. A guest is reaped,
so it has nothing to resume.

- **Three link states** on `connection.link`. `setConnected` →
  `connected`; a drop → `reconnecting`; give-up → `dropped`.
- **Exponential backoff over ~60s** (`services/websocket.ts`:
  `1·2·4·8·16·30·30s`, replacing the fixed 5×2s) so users ride straight
  through a server restart without acting.
- **Command send is disabled while down, never queued** (`CommandBar`
  reads `connection.link`; a buffered command replayed into a resumed
  world is a foot-gun).
- **On give-up (real user):** `ReconnectBanner` shows over the preserved
  cockpit; its button (`websocketClient.reconnectNow()`) resets the
  backoff and resumes. Never a silent bounce to a dead page.
- **Routing to the start screen** — the two cases with nothing to
  resume: an unauthenticated session (re-auth needed) and a **dropped
  guest** (avatar reaped; the store also clears identity).
- A **successful reconnect is seamless** — subscriptions already
  re-issue on `connection-established` and welcome-scene-on-reconnect is
  already suppressed; the client re-enables input and clears the banner.
- **Intentional teardown** (logout / leave-world) sets
  `intentionalDisconnect` so `onclose` skips auto-reconnect — the caller
  drives the phase itself.

## The honest-state primitives

`components/ui/Figure.tsx` + `UnbuiltGround.tsx` — the shared primitives
for the one convention that cuts across every surface that shows a
number (`docs/design_handoff/CONVENTIONS.md` #1):

> **Never render a figure the server did not send**, including "just for
> now."

> ⚠⚠ **The nightly wipe this paragraph argues from did not exist** —
> found 2026-08-13 while building Arrival. No cron, no CI job, no
> script, and deployment.md documents durable Mongo
> Atlas persistence. The assumption had propagated into three documents.
> It is scheduled to be built (the record-layer build), and until it is,
> **no surface may state it**: the front door renders a reset notice
> only when `/auth/status` reports a `resetPolicy`, which is absent by
> default. See [the front door](#the-front-door).
>
> ⭐ The lesson outlives the fix: *a factual premise stated once in a
> governing paragraph gets cited, not re-checked.*

The demo wipes nightly, which buys latitude on *persistence* and none on
*figures*: a plausible fake is indistinguishable from a bug, and this
product's central claim is that its numbers are real. Three states must
look nothing alike:

| State | Rendering | Reason shown |
|---|---|---|
| `live` | a number. no decoration. the only state that shows one | none — a live figure has nothing to explain |
| `empty` | `—` in the muted foreground | **yes** — a real zero deserves a reason |
| `unwired` | hatched ground, dashed border, `╌╌` where the value goes | **yes** |

⭐ **The union is the deliverable.** One component with a *required*
discriminated `figure` prop, not three components and not an optional
`value`:

```ts
export type FigureState =
  | { readonly state: "live";    readonly value: string }
  | { readonly state: "empty";   readonly reason: string }
  | { readonly state: "unwired"; readonly reason: string };
```

You cannot render a figure without naming its state; `empty` and
`unwired` cannot omit their reason; and **`unwired` has no `value` field
at all**, so the convention is enforced by the type checker rather than
by vigilance at every call site. Three separate components would leave
`<span>{n}</span>` as the path of least resistance — and the fourth
state the convention warns about, a plausible fake, is exactly what the
path of least resistance produces. `Figure.test.tsx` asserts the
compiler's refusal with `@ts-expect-error`; CI enforces it through the
client's `build:types` job.

Tokens: `hatch` / `hatchStrong` for the 135° stripe, `info` for the
`╌╌`, `fgMuted` for the `—` and the reason, `accent` for a live value.

Two decisions worth keeping:

- ⚠ **No stamp.** The reference art's `live` / `empty` / `not wired`
  chips are documentation labels for its three example cards; the
  convention's own wording is "a reason, **not a stamp**". The hatch is
  the stamp and the reason is the words.
- ⚠ **No `color-mix(in oklab, …)`**, which the reference art uses
  throughout. `hatch` / `hatchStrong` are precomputed per ground —
  a marble hatch is not an ink hatch lightened, mixing against a
  transparent stop composites over whatever is behind, and precomputing
  makes each theme state what its hatch actually looks like. It is also
  readable in jsdom, so the hatch is testable rather than intended.

Accessibility carries the honesty to a screen reader, to which a dashed
border is invisible and `╌╌` is noise: `role="group"` plus an
`aria-label` that says "not wired" / "none" in words, with the reason.

⚠⚠ **The chip's `title` IS that `aria-label`, not a second copy of it.**
It was once built separately with `not wired` hardcoded for every
non-live state, so an `empty` chip — a figure the server *did* answer,
with nothing — told every pointer user it had no endpoint. That is the
one collapse this whole vocabulary exists to prevent, and it landed in
the attribute that, on a chip, is the *only* place the reason appears:
the glyphs `—` and `╌╌` do not spell out which state they mean.
`Figure.test.tsx` now asserts the two attributes are the **same
string** rather than re-asserting the words, because two descriptions
of one thing is the defect and equality is the form that cannot drift.
⭐ Found by rendering the design mock beside the live chrome — the
aria-label tests covered all three states correctly and nothing read
the other attribute.

**Two carve-outs, neither belonging to these components.** *Prose never
hedges* — a room description carries no engineering stamp, because
breaking the fiction to report an engineering fact is the wrong trade in
the one place the game is supposed to be a world; if a thing cannot be
described yet, it is not in the room yet. And *commands refuse honestly*
in the machine voice (`cockpit style theme default` naming the three it
does know), not through a hatched widget.

### The `variant` axis

`Figure` carries a layout axis orthogonal to its state union:

| Variant | Shape | Used by |
|---|---|---|
| `card` (default) | label over value over band, with a visible reason | the original Build A block |
| `chip` | one 30px line, band suppressed, reason via `title` + `aria-label` | the widget shelf |
| `row` | full-width, label left / value right, reason beneath | the connection popover |

⚠ **A variant enum, not a second component and not a `styled(Figure)`
override.** A second component reopens the `<span>{n}</span>` hole the
union was shaped to close — the constraint is that no shelf row may
print a value outside `Figure`, and the moment there are two ways to
render a figure there are three. A `styled(Figure)` override with
descendant selectors would put the primitive's internal DOM shape in the
consumer's hands, so a refactor of `Figure` would silently break the
shelf's layout with no type error. `card` stays the default, so
`Figure.test.tsx` passes unmodified including its `@ts-expect-error`
compiler assertions.

⭐ **The widget shelf is the first consumer**, and being that consumer
is how the primitive learned it needed the variant axis at all. (The
prior note here read "ships with no consumer" — the accepted Wave 1 risk
that a primitive with no consumer can drift. Build B closed it.)

## What this build does NOT add

- **No client-side persistence / localStorage.** Everything that matters
  is server-authoritative behind the session cookie; nothing in scope
  needs persistence the cookie doesn't provide. The pre-auth /
  device-local tier is a real future concept (per-device perf toggles, a
  pre-auth theme cache) **deferred to the slate** with no v1 content.
- Search, mode switcher, the public read-only surface, the declarative
  mode/manifest model, author mode / CMS — all later cycles (see the
  client-shell slate).
- **⛔ The held-commands queue.** Cut with a reason, recorded here and in
  the slate so the art's 6D is not later read as an unmet promise. See
  *the dropped row* above: an offline queue with ordering, expiry and
  replay-safety questions is a real feature deserving its own
  requirements. The dropped row's *commands sent now will not arrive*
  is what it would replace.
- **⛔ Touch gesture infrastructure beyond the one pull-down.** The grab
  toggles on tap; there is no swipe navigation, pull-to-refresh or edge
  gesture. A full gesture system (velocity, rubber-banding, partial
  states) is its own build, and the art's *arrives when you drag* is
  satisfied by an animated disclosure.
- **The mobile Arrival path** (front door, intake, lounge, character
  select) — Wave 2, with its own `- Mobile` art. `CharGenStage` is
  untouched here.
- **The mobile live client / play surface** — the two feeds, the card
  feed and its hold policy, focus chain, filters, routing, prompts.
  Wave 4, which is where the feed itself is owned, and where the
  `WorldLayout` rail's phone-width overflow belongs.
- **A frames-behind figure.** Still unmeasured — no server sequence
  number, and unlike round trip there is no existing protocol for it.
  Stays hatched, with a reason naming what is actually missing.

## History

- **Guest build homed on `Login`** (MR !54 review): the randomized
  guest-avatar build started life as `EnrollController.mintRandomGuestAvatar`
  (next to the char-gen `commit` it mirrors) but moved onto `Login` — the
  guest-mint site — reading the rosters + `validSexSet` back from
  `EnrollController` via a lazy import. In the same pass the
  `GUEST_RESERVED_WORD` + guest-name generation moved from `Avatar` to
  `Login`, the hardcoded guest-surname fallback list was deleted (real
  `common` NameBank only), and `PRONOUN_LABELS` was colocated with the
  `Pronouns` enum in `@saxonberg/types`.

## ⭐ The card surface, and where the honest-state convention lands on it

The right column is a **feed of cards** — the `Inspect · Who's Online ·
News · Wiki` switcher is gone, and those three are catalogue rows now.
See [card-surface.md](./card-surface.md).

Three of this doc's own conventions do real work there:

- **Honest state, on a new axis.** A **static** card renders its
  timestamp (*taken 14:32*) and a refresh control; a **live** card
  renders neither. A static card that looked live would be a lie, and a
  refresh button on a live card is a bandage over a wake that does not
  fire — *and, worse, it is how nobody finds out.*
- **Every clickable previews exactly what it sends.** A refresh
  control's label, its `aria-label` and its payload are the card's own
  `key` — the normalized command that produced it — so the preview and
  the send are literally the same string. The pin sends
  `cockpit card pin <id>` and mirrors the server's answer rather than
  toggling locally.
- **A section that does not apply is absent, not hatched.** The card's
  action row renders only subject-afforded verbs, so on an ordinary
  object with no `commandContributions` it is absent entirely.

⚠ **The wiring lives in `App`, above the mode registry** — the third
position it has occupied, and the one with no fourth. It sat in
`CardFeed` (desktop only), which left the whole phone card surface
dead; it moved to `WorldLayout`, which fixed the phone and left `build`,
`chat` and `watch` — different layout components entirely — with the
same defect. Every unit test passed through both, because **a component
test proves rendering, never wiring**; the guard is over the source.

⚠ **The summoned-overlay tier is called `panel`, not `card`.**
`SettingsPanel`, `SocialNotificationsPanel`, `LivestreamPanels`,
`CmsDiagnosticsPanel` dock beside the layout and close when you close
them; they have no identity, no lifetime and no pin. Renaming them
`*Card` during the `pane`→`card` sweep would have been a lie about what
they are.

