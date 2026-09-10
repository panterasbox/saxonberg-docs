# The cockpit

The client front-of-house layer: **one verb, two axes, a card set, and a
client that is a pure view over all of it.** The governing principle:
**the client owns zero command semantics.** Every mode switch, every
arrangement recall and every scope change is a real command on the wire
— replayable, scriptable, attributable, visible to a stream overlay.

Built against the binding composition grammar
(never-blind, fixed-chrome/fluid-content, hierarchy-encoding splits, the
no-modal rule).

## ⭐ One verb: `cockpit`

The cockpit's controls had scattered across **three** top-level verbs
that each grew independently — `layout`, `style`, and `mode` — and every
one of them wrote the same `cockpit.*` clientState keyspace. Adding a
fourth for the new axis would have compounded the scatter rather than
noticing it.

```
cockpit                       report everything in effect
cockpit mode <name>           what you are here to do
cockpit layout <name>         the card arrangement inside that mode
cockpit cli [id] --prefix …   prefix one command line's bare input
cockpit shelf list|pin|unpin|first  which figures ride the top bar
cockpit style <sub> …         appearance
cockpit card pin|dismiss …    override what holds a card open
```

`layout` and `style` are **removed** as standalone verbs, not kept as
aliases: one name per thing, and two names for one thing is a cost that
never stops being paid. `mode` is absorbed as **`cockpit cli`** — it
always wrote `cockpit.inputModes`, so it was a cockpit concern wearing a
top-level verb, and its old word was needed for the activity axis.

The three controllers survive as **per-subcommand controllers**
(`controller:` on each subcommand), so the consolidation was wiring, not
a rewrite.

### `cockpit shelf` — the widget shelf's pin surface

```
cockpit shelf list            the whole catalogue, and what is pinned
cockpit shelf pin <row>       add a figure, at the end
cockpit shelf unpin <row>     take one off
cockpit shelf first <row>     move it to the front
```

Rows: `play` `renown` `skill` `make` `coin` `status` `time` `online`
`docket`. Persists to **`cockpit.shelf`**; the surface it drives is
documented in [client-shell.md § The widget shelf](./client-shell.md).

### ⭐ `first`, and the glance-line

The shelf has always had an **order** and, until the phone chrome, no
way to change it: `pin` appends, `unpin` removes, and what you ended up
with was whatever pin-sequence you happened to type. That was survivable
on a desktop bar that wraps. It is not survivable on a narrow one, where
the bar shows only the **head** of the shelf — `shelf.slice(0, 3)`, the
glance-line — and the rest lives behind a pull-down. So *choosing what
rides a phone's bar* **is** *reordering the shelf*.

The alternative was a second clientState key naming the bar rows
explicitly, and it was rejected for the reason two parallel lists are
always rejected here: an unpin that forgot the glance key, a glance row
that is not on the shelf, and a `list` output that cannot say which is
true. **One ordered list has one answer.** It is also a desktop
improvement — shelf order had been unchangeable since the shelf shipped.

⚠ `first` on an **unpinned** row pins it at the front, because *"put
this on my bar"* is one intention and making a player type `pin` then
`first` would be two commands for one thought. `first` on a row already
at the head does not write at all — an identical array pushed back would
re-save and re-broadcast for nothing.

Naming: `first` over `promote`/`move`, because it names the resulting
**state** rather than the operation, and it reads correctly in the
machine voice — *coin moved to the front of the shelf*.

⚠ It is a fourth **action** on the existing positional slots, not a new
subcommand: `cockpit shelf` has already spent the one level of nesting
the framework has. So the YAML gained help text and an example and
nothing else, and the refusal that names the known actions had to grow
to four — an action a refusal does not name is a vocabulary a player
cannot discover.

⚠ The count (three) is a **client** rendering decision, not a server
constraint. The server does not know how wide a bar is; a shelf shorter
than three simply shows what it has. See [client-shell.md § The mobile
bar](./client-shell.md).

⭐ **The default pins only `play`, `renown`, `skill`** — the three that
are actually wired. *Never default-pin a widget that does not do
anything yet*: the other six render their honest not-wired state, and a
new player's first impression should not be six dead boxes. They are one
`cockpit shelf pin` away, and `cockpit shelf list` is how a player
discovers them — which is why `list` prints the whole catalogue rather
than only what is pinned.

Mirrors `cockpit card` in every respect — `list` / verb / `<id>`, two
positional slots (`action`, `row`) dispatched in the controller, and a
refusal in the machine voice naming the known set. Four artifacts: a
YAML `subcommands:` block, `CockpitShelfController`, a two-line seed,
and a `clientStateSchema` entry.

⚠ **Pinning is a command and not a client toggle**, which is the whole
reason it is here at all. The status bar shipped in the same build
advertises one axiom — *every click sends a command, and the interface
shows which*. A pin affordance that mutated local state while the status
bar previewed a command would falsify the one claim that bar exists to
make. (The second reason is ordinary: the cockpit keyspace is already
server-owned, and a divergent persistence path for one preference is
drift.)

⭐ **`list` reports PINNED-NESS, never live-or-hatched.** The honesty
vocabulary — which rows show a real figure and which show their reason —
is **client-side**, because hatched-ness is a property of the client's
wiring rather than of the server's capability. The server cannot know
which of the fields it sends the client actually paints; a verb printing
a guess would be exactly the confident wrong answer the honest-chrome
build exists to eliminate. Modelling it server-side would mean the
server holding a second source of truth for something only one side
observes.

⭐ Identity and connection refuse as *unknown shelf row*, because they
are **not rows** — a stronger guarantee than a protection rule.

### ⚠ The style tree rides positionals, not nested subcommands

`cockpit style theme high-contrast` looks like a subcommand of a
subcommand. It is not, and cannot be: `SubcommandDefinition` has no
`subcommands` field — **subcommands are one level deep framework-wide**,
and nesting them would be a change to the dispatch chain.

So the style settings ride generic positional slots (`styleSub` / `a` /
`b` / `c`) and `StyleController` dispatches on them — the same shape
`style channel <key> color|clear` already used for its own third level.
The player-facing syntax is unchanged.

⚠ **The cost, and the reason `StyleController` has an explicit
`default:`** — an unknown setting no longer reaches the matcher's
`unknown-subcommand` refusal, because `cockpit style` *is* the
subcommand. Without that default, `cockpit style bogus` would fall out
of the switch and the command would silently succeed while doing
nothing, which is a worse failure than an error.

### ⭐ The input-mode exemption is now a rule

`CommandApi.applyInputMode` hardcoded the literal `'mode'` so that mode
management always reached the interpreter un-prefixed. That literal is
now **`cockpit`**, and the rule it expresses is finally sayable:

> **Interface control is not world input.** A bar scoped to a chat
> channel still steers its own cockpit.

This widens the exemption from one verb to one verb's whole subtree, and
that is correct by construction — everything under `cockpit` is
interface control, so there is nothing beneath it that *should* be
prefixed. A narrower exemption would strand a player who scoped a bar to
gossip: they could not un-scope it without knowing the `/` escape.

⚠ Tested in **both** directions. That cockpit subcommands run
un-prefixed is half of it; that ordinary verbs typed in that bar are
*still* prefixed is what makes it an exemption rather than a hole.

## The two axes

They answer different questions, and conflating them is what the old
`cockpit.layout` did.

| Axis | Question | Key |
|---|---|---|
| **mode** | what am I here to do | `cockpit.mode` |
| **arrangement** | how do the cards sit inside that | `cockpit.arrangements` |

### The mode axis — the front doors

`CockpitMode` / `COCKPIT_MODES` in `@saxonberg/types`: `chat` · `play` ·
`watch` · `build` · `govern`.

⚠ **A mode is a view, never a gate.** Everything runnable in `play` is
runnable in `build`. A mode that forbade a verb would be a permission
system wearing a UI costume, with its checks in the wrong layer, no
audit trail and no way for a player to discover why a verb vanished.
"Study mode shouldn't allow combat" is a seductive one-liner and it is
the wrong layer for that idea.

This is enforced, not merely asserted: a test resolves a verb in all
five modes, and a source scan asserts **nothing outside the cockpit's
own four files so much as reads `cockpit.mode`**. A gate has to read the
mode to enforce it, so "who reads it" is the complete list of places one
could hide.

A mode owns which arrangements ship, which one you land in, and which
card kinds may be summoned. Nothing else.

⚠ `govern` ships as a peer of `build` rather than a tier inside it. The
seeding slate writes the pair as "the Build / Govern ascent", which
reads as one progression; they are two values because a front door is a
front door. If `govern` turns out to belong *within* `build`, that is a
one-line edit to `COCKPIT_MODES`, not a redesign — flagged deliberately
rather than silently resolved.

### ⚠ What the arrangement axis ships, and what it does not

**Storage and vocabulary — not behaviour.** `cockpit layout save`
captures the cards that are open, by catalogue name; `recall` sets the
active arrangement in `cockpit.arrangements`. **Nothing opens or closes
a card in response**, on either side of the wire: no client code reads
`cockpit.arrangements` today.

That is deliberate scoping rather than a gap to be patched. The consumer
is the client rebuild, and inventing a restore path before that client
exists is how the same feature gets built twice. But it is stated here
because the surface *looks* complete: `save` reports a card count,
`list` shows names, `recall` succeeds, and none of it moves a card.

⭐ **The unstated half is who acts on a recall.** The server has no way
to tell a client to open a subscription — the client always initiates —
so either that mechanism gets invented, or the client reads the
arrangement and opens the named cards itself. The second works with what
exists and keeps the split the card catalogue already established: the
**client initiates, the server owns the vocabulary**.

⚠ Saved arrangements are capped per mode
(`MAX_SAVED_ARRANGEMENTS_PER_MODE`). A save writes a player-chosen key
into a persisted map, so uncapped it is unbounded growth of the player's
own document — self-inflicted, which is exactly why it is easy to leave
open. Overwriting an existing name is always allowed: it adds no key.

### The arrangement axis — savable, not a frozen list

⚠⚠ **An arrangement is not a closed vocabulary.** `cockpit layout` used
to validate against `LAYOUT_NAMES`; it cannot any more, because
arrangements are *savable* — a player composes and names one. It
resolves against **the active mode's shipped defaults plus that player's
saved arrangements**, which is why `LAYOUT_NAMES` was *replaced* by
per-mode defaults rather than promoted.

```
cockpit layout <name>          recall
cockpit layout list            what is available here
cockpit layout save <name>     name the current arrangement
cockpit layout forget <name>   drop one you saved
```

`COCKPIT_ARRANGEMENTS` holds the shipped floor; the first entry of each
list is that mode's default. `watch` ships two (`viewer`, `streamer`).

- **Scoped to a mode.** A name living in a *different* mode is refused
  **with that mode named**. "Unknown arrangement" would be a lie to a
  player who has one by that name one door over, and silently switching
  their mode to find it would be worse — a recall is not a mode change.
- **No silent shadowing.** A saved name may never shadow a shipped
  default, and the collision is refused at **save** time, where it can
  still be explained, rather than resolved at recall time in favour of
  whichever list happened to be checked first.
- **Bounded.** Names are player input that become persisted map keys and
  get rendered back: 32 characters, no spaces.
- **Remembered per mode.** `cockpit mode chat` → `cockpit mode play`
  returns you where you were.

`cockpit mode <name> [arrangement]` sets both at once — which is what a
Views-menu item means, and why the menu stays one click = one
previewable command.

### ⚠ The legacy migration is a MAPPING, not a rename

The five old `cockpit.layout` values were really *a mode plus that
mode's arrangement* flattened into one string, and every player who ever
ran `layout builder` has one persisted.

| legacy | → mode | → arrangement |
|---|---|---|
| `world` | `play` | default |
| `forum` | `chat` | default |
| `livestream-viewer` | `watch` | **viewer** |
| `streamer` | `watch` | **streamer** |
| `builder` | `build` | default |

The livestream row is the one that matters: **two legacy values collapse
into one mode with different arrangements**, so the mode column alone is
lossy and only the arrangement column keeps them apart.

Resolved **on read** (`getCockpitMode` / `getCockpitArrangement`), so
nothing has to sweep stored Avatars and a player who never returns costs
nothing. Two details that are easy to get wrong:

- `cockpit.mode` defaults to **`null`, not `'play'`**. A `'play'`
  default would answer for a legacy player before the migration could,
  and the migration would silently never run.
- The legacy arrangement is **scoped to the mode it migrated to**.
  `livestream-viewer` may answer for `watch`; it must not answer for
  `play`, or one mode's vocabulary leaks into another.

`snapshotClientState` emits both axes **resolved**. A raw snapshot would
hand a legacy client `cockpit.mode: null` and make the *client* decide
what that means — the one thing the client may not own.

### ⚠ `cockpit.layout` survives as a compatibility projection

The shipped client still swaps its whole frame off `cockpit.layout`,
keyed by `LayoutName`, and repainting it is a separate cycle. So the
server keeps that key painted from (mode, arrangement) via
`LEGACY_LAYOUT_FOR` / `legacyLayoutFor`.

**It is marked to die with the client overhaul.** The two real axes are
mode and arrangement; without the projection the cockpit would go blank
the moment arrangements stopped being layout names, which is the only
reason it exists.

Not total, and cannot be: `govern` has no legacy layout at all, and a
player-saved arrangement has none by construction. Both fall back to the
mode's first mapping — the honest answer, since the old client cannot
render them and renders the nearest thing it has.

## Cards: pinned, or aged out

See [card-surface.md](./card-surface.md) for the card set itself.
The cockpit's half is the override verb: `cockpit card pin|dismiss|auto|list`.

⭐ **Pinned or unpinned is the WHOLE lifetime.** The five hold
conditions this verb used to override are gone: four were spatial and
each cost a wake to fire, and `unanswered`'s guarantee — *nothing still
actionable ever leaves* — moved onto this axis, where a prompt card
opens pinned and auto-releases when answered. `auto` hands the decision
back to the catalogue's own default rather than to a condition.

⚠ **The pin is a command, and its answer is mirrored, never assumed.**
`cockpit card pin <id>` can be refused, so the client renders the
server's `pinned` value off the `card-pinned` envelope rather than
toggling locally — a local toggle shows a pin that is not there, and it
does not survive the tab.

⭐ `cockpit card list` says which cards are LIVE. Liveness is the axis a
player cannot otherwise observe: a static card shows a timestamp and a
refresh, a live one shows neither, and without the word here the only
way to tell them apart is to watch one fail to update.

⚠ **Dismiss is not on the pin control.** It is a different intent —
*drop this even though it still holds* — and putting it in the same
cycle would make one accidental extra click destroy a card the world
says is still relevant. The pin cycles pin ↔ auto; dismiss is its own
subcommand.

## ⭐⭐ A mode switch opens its arrangement, server-side

`cockpit mode <name>` resolves the saved arrangement **on the server**
and pushes the card set. The client sends one command and renders what
arrived.

This is what keeps *the client owns zero command semantics* literally
true. The alternative — the client replaying `cockpit card open <name>`
per card — puts the meaning of an arrangement, and the card ORDER, in
the client, and costs a round trip per card.

⚠ The cost is accepted knowingly: **the server now holds view state per
player.** It already holds `cockpit.layout`, `cockpit.mode` and
`console.tabs`, so this is the same seam widened, not a new one.

The rules `applyArrangement` follows:

- **Subject cards are skipped.** An arrangement is a statement about a
  workspace; a card about a particular person is a statement about a
  moment, and restoring one next week would restore an answer to a
  question nobody is asking.
- **An already-open card is left alone**, rather than closed and
  reopened — the reopen would lose its pin and its hold.
- **A shape-opened card is never closed.** The player opened it; a mode
  switch is not a licence to throw it away.

⭐ **It also runs on LOGIN** (`Avatar.enter`), not only on a mode or
layout switch. It was called from those two controllers alone, so a
player who logged straight into `build` saw an empty feed until they
switched modes and switched back — an arrangement that only applies when
you change your mind is a workspace you cannot simply return to.

⚠ `SHIPPED_ARRANGEMENT_CARDS` is keyed by **(mode, arrangement)**, not
by mode alone: `watch` ships two arrangements and one flat list per mode
cannot express them. The mode-only shape survived only while the map was
empty.

### ⚠⚠ The `pushed` flag, not an inference

A result the server pushed carries `pushed: true`, and the client adopts
a card only for a handle it does not know **and** that flag. "A handle I
do not know" was tried and is wrong twice over: the chrome's own named
cards echo `card` too, and a result arriving after the client's own
unsubscribe — which React's double-mount produces on every dev page
load — looks unknown as well. Both showed up live as spurious cards, and
no unit test could see either, because the adoption path only fires for
an envelope a test would have to mint by hand.

## The old layout axis (still the client's shape)

Layout is per-player UI state, identical in kind to `console.tabs` — so
it is a `clientState` key, not a client-local field. The client registry
below is keyed by the legacy `LayoutName` and is what the compatibility
projection feeds.

- **No auto-switch.** `cockpit layout` / `cockpit mode` are the *only*
  ways to change what is on screen — typed, or sent by the "Views" menu.
  The `forum` verb stays pure CRUD (you reach the board view via
  `cockpit mode chat`); the standalone `?surface=cms` takeover is
  retired. No domain verb, URL, NPC, or item flips it.

### Client registry

`packages/client/src/layouts/` holds a `layout → component` registry
(`LAYOUT_REGISTRY`, a `Partial<Record<LayoutName, LayoutDef>>`). `App`
reads `clientState['cockpit.layout']` and renders
`LAYOUT_REGISTRY[layout] ?? LAYOUT_REGISTRY.world`. The transient
client-local `mainView` flip and the `recognizeForumNavigation` watcher
are **deleted** — all view-switching lives in the one server axis.

- `LayoutProps` is the bundle `App` threads to the active layout (frame
  buffer, command-send/preview/click handlers, prompt handlers). Each
  command bar owns its own local input draft and submits its `barId`;
  preview/flash live in the status bar, not in a bar.
- The **"Views" menu** (`components/ViewsMenu.tsx`) previews/sends
  `cockpit mode <mode> <arrangement>` per the click model and marks the current layout. It +
  the Settings affordance live in the **single top `Frame` bar**
  (`components/frame/Frame.tsx`) alongside the connection chip, identity
  and the widget shelf — one
  fixed chrome row, not two (the separate `ChromeBar` row was folded into
  `Frame` so the content area reclaims the vertical space).
- An **always-on minimum** (the Frame bar, a command bar) renders in
  every layout, so the player can always type `cockpit mode play` to leave.

### The five layouts

| Layout | Shape (canonical split) | Notes |
|---|---|---|
| `world` | terminal + a right rail with an **Inspect \| Who's Online** card switch (Single + rail) | the classic cockpit; the rail card is chosen by the `rightCard` store slice (inspection detail vs the social-inspection roster) |
| `forum` | board view + chat sidecar | the old `mainView === 'forum'`; live-scene peek keeps it never-blind |
| `livestream-viewer` | video embed (focal) + game terminal + chat rail | see below |
| `streamer` | a **control deck** (focal) + game terminal + a widened chat rail | `StreamerDeck`; see below |
| `builder` | CMS editor (dominant) + glance terminal rail (Monitor) | the CMS re-homed in-session |

`LivestreamViewerLayout` and `StreamerLayout` share `LivestreamPanels`
(the game terminal + chat rail + bar); only the focal card differs (the
viewer's video embed vs the streamer's control deck), and the streamer
passes `railWide` so its chat — the broadcaster's lifeline — gets more
room. The shared side rail is the tokenized `SideColumn` primitive
(`tokens.rail`, with a `$wide` variant for the builder/streamer); the
focal split is a real `tokens.ratio.focal` (~62/38) so fixed-ratio
content (the 16:9 embed) is deliberately large and letterboxed within
its share rather than incidentally sized.

### The streamer control deck

`StreamerLayout`'s focal is `components/StreamerDeck.tsx`, **not** a stats
readout: while live a broadcaster mostly needs quick-action shortcuts to
the commands they fire at the community (moderation, rewards/penalties,
community prompts, broadcast state) plus a small live readout (viewers,
uptime). The deck is the *frame* for that — a live strip + grouped cards
of command shortcuts. The streamer command set isn't built yet, so the
chips are an **honest scaffold**: each previews the command it *would*
run in the status bar on hover (the same learn-the-CLI affordance) but is
dimmed and doesn't send until its verb ships. Per the through-line the
client still owns zero command semantics — every chip is just a
command-bus affordance.

## Livestream-viewer + the watch-driven embed

The focal video is now **per-viewer, driven by the `watch` verb** — not a
global operator-curated list. The whole `broadcastSources` path
(`StreamSourceApi`, `livestream.broadcastSources`, the `stream-sources`
envelope, `Events.StreamSourcesChanged`, `Application.wireBroadcastSources­Push`,
the welcome-snapshot `broadcastSources` field, and the multi-source picker)
is **retired**. See [streaming.md](./streaming.md) for the `tune`/`watch`
surface; this section is just the embed's cockpit half.

- **Server-authoritative per-viewer state** — `watch <target>` resolves the
  embed shape and writes it to the transient `cockpit.watch` clientState
  (`WatchTarget | null` in `@saxonberg/types`), pushed to the client via the
  `setClientState` → `pushClientStateUpdate` seam (the `cockpit.inputModes`
  / `ModeController` precedent, no `save()`). `watch off` → `null`. The
  client owns zero command semantics — it mirrors the server state and
  renders the iframe.
- **`WatchTarget`** is embed-shaped: `{platform:'twitch', channel}` |
  `{platform:'youtube', videoId}` | `{platform:'youtube', channelId}` (the
  `@handle`/`UC…` durable form, rendered `live_stream?channel=<channelId>`).
- **Client** — `components/embed/StreamEmbed.tsx` takes a single
  `WatchTarget | null` and renders **one** sandboxed iframe (no picker) or
  an empty placeholder. `LivestreamViewerLayout` reads
  `clientState['cockpit.watch']`. The Twitch iframe's `parent` is
  `window.location.hostname` (correct-by-construction, never hard-coded);
  the YouTube iframe is `youtube.com/embed/<videoId>` (videoId arm) or
  `youtube.com/embed/live_stream?channel=<channelId>` (channelId arm). The
  chat terminal allowlists the relay topics (`speech.relay`); the game terminal is the complement —
  client-side filters over the one shared frame buffer, no ingest-time
  routing.

## Builder = the CMS re-homed

`BuilderLayout` mounts the existing `CmsSurface` (its `Screen` restyled
from `100vh/100vw` to `flex:1; min-height:0`) beside a glance terminal
rail. The WebSocket session is **live** the whole time — the standalone
`?surface=cms` full-screen takeover + its WS guard are deleted, so the WS
always connects. The `CmsApi`/`CmsLogic` REST surface +
`CmsSession.runAsSessionPlayer` attribution bridge are reused **unchanged**
(a client re-mount, not a CMS rewrite). The dev account-menu launcher
sends `cockpit mode build` over the bus instead of opening a tab.

## Server-authoritative input mode (per bar)

The last client-only command behavior, moved server-side — command-bus
primacy made absolute.

- **Schema** — `cockpit.inputModes` is a `clientStateSchema` entry, a
  `{ barId → prefix }` map (default `{}`, object-of-strings validator),
  marked **`transient`**. Input scoping belongs to the live session, not
  the character: transient keys live in a separate, non-persisted
  in-memory `_transientClientState` slot (a per-key `transient` flag on
  `ClientStateSchemaEntry` routes get/set/snapshot there), so they never
  reach the Hydrator and **reset to `{}` on a fresh login**. The `barId`
  is thus an in-session routing handle, not a saved cross-session
  vocabulary.
- **`barId`** rides the inbound `command` message and threads through
  `ExecuteCommandOpts` → `CommandContext` → the dispatch. A submission
  with **no `barId`** (an affordance click, a script, programmatic
  dispatch) is deliberately un-moded; only a real bar submission carries
  its `barId`.
- **The interpreter prepend** (load-bearing, on the hot path):
  `CommandApi.applyInputMode(rawText, modePrefix)` is a **pure, total**
  pre-tokenize step — (1) no prefix → verbatim no-op; (2) leading `/` →
  strip the slash, run raw; (3) the `mode` verb itself is exempt;
  otherwise prepend (`chat` + `hello` → `chat hello`). `executeCommand`
  hooks it behind `barId !== undefined && interactive && !forced &&
  isHasInteractive`, looking up `cockpit.inputModes[barId]`. Kept in
  `CommandApi` (not `msh`) so the tokenizer stays Stuff-unaware; the
  per-bar lookup is at the call site. The echo reflects the **dispatched**
  text.
- **`mode` verb** (`cmd/shell/mode.yaml` + `ModeController`) edits one
  line's entry — `cockpit cli --prefix "…"` sets, `cockpit cli --clear` clears, and **bare `cockpit cli` REPORTS**. ⚠ The predecessor conflated report and clear: a bare invocation wiped the prefix, so checking what was set destroyed it.
  The target resolves `model.bar ?? context.barId ?? 'main'`: a typed
  `mode` defaults to the bar it was submitted from (`context.barId`), and
  the explicit **`--bar <id>`** option names a bar for an affordance —
  which dispatches *un-moded* (no wire `barId`, so preview == send) and so
  can't rely on `context.barId`. The target rides in the command text
  (`cockpit cli stream-chat --prefix chat`), keeping the status-bar preview honest.
  Because modes are transient the commit is just **write→push** (no
  `save()`).

### Client (display-only)

The client no longer wraps input. The `inputMode` store slice is gone.

- Each `CommandBar` is bound to a **`barId`** (distinct per layout bar:
  `world` / `forum` / `stream-game` / `stream-chat` / `builder` /
  `chargen`), owns its base draft **locally** (so multiple bars coexist),
  and submits `(tail, barId)` verbatim. A moded bar renders its prefix as
  an **inline, uneditable `<span>`** styled identically to typed text
  (read from `cockpit.inputModes[barId]`); it **hides when the input is
  exempt** (`/` or `mode`), so the bar always shows what dispatches.
  Closing the mode is a small `✕` at the bar's edge (+ Esc), both sending
  `cockpit cli --clear` from that line. The chat sidecar's "talk here" sends
  `cockpit cli --prefix "chat <handle>"` from the forum line.

### The preview surface + click model

Under per-bar mode, preview cannot live *in* a bar (a moded bar would
prepend its prefix, so the preview would lie). It lives in the always-on
**status bar** (`components/frame/StatusBar.tsx`, store-backed
`ghostPreview`/`ghostFlash`) along the bottom of the window. Hover → the
exact command; **click → run it un-moded** (no `barId`, so preview
equals send); **shift-click / right-click → copy** to the clipboard (the
explicit "make it mine" path, replacing shift-click-loads-a-bar, which
has no honest target under N bars).

⚠ **`GhostCommandLine.tsx` is retired** — it had two render sites, and
two surfaces showing what a click would send can disagree, which
discredits the axiom rather than merely looking wrong. There is now
exactly ONE preview surface, guarded by a source scan. See
[client-shell.md § The status bar](./client-shell.md).

## The summoned-panel tier

The no-modal rule (composition law 6) routes would-be modals into two
tiers that both keep a terminal on screen: the **layout tier** (above)
and the **summoned-panel tier** — transient panels that dock *beside* the
current layout's terminal (the inspection card is the existing proof).
This build establishes the summoned-panel tier as a first-class mechanism:
a `summonedPanel` store slice (`openPanel`/`closePanel`) and an
`App`-level `ContentRow` that docks the panel beside the active layout.

⚠ **It is a PANEL, not a card**, and the word matters. A card is a
container for structured content in the feed, with an identity, a
lifetime and a pin; a summoned panel is chrome that docks beside the
layout and closes when you close it. The card build renamed these
deliberately (`SettingsPanel`, `SocialNotificationsPanel`,
`LivestreamPanels`, `CmsDiagnosticsPanel`) rather than sweeping them
into `*Card`, which would have been a lie.

**Settings** is the first consumer (`components/settings/SettingsPanel.tsx`):
a non-modal side panel whose controls each **send the real command**
(`settings set …` / `var set …`, un-moded) per command-bus primacy — no
client-only settings state. Its notification section consumes the
(separately-built) notification-settings backend when present and
**degrades to a "coming soon" placeholder** when absent (the case on this
branch). Opening the card is client UI state (like the inspection card),
not a `layout` switch — it coexists with the current layout.

## The client frame renders from both axes (Wave 6)

`cockpit.layout` shipped as a **compatibility projection painted from
(mode, arrangement) so the shipped client keeps working** — explicitly
meant to die with that client. Wave 6 is where it died.

`packages/client/src/layouts/modes.ts` holds `MODE_REGISTRY`, keyed
`(mode, arrangement)`, and `resolveMode`, which is the **one** place the
legacy key is still read.

⭐ **`watch` is why the axes are two and not one.** `livestream-viewer`
and `streamer` were never different activities — they are two
arrangements of one, which is why `LEGACY_LAYOUT_MIGRATION` is a mapping
rather than a rename, and why the collapse is lossy in the mode column
with only the arrangement column keeping them apart. No single-axis
registry can express that; `modeResolution.test.ts` asserts it directly.

⚠ **The legacy key is read exactly where the SERVER reads it.**
`cockpit.mode` defaults to `null` deliberately, so `getCockpitMode()` can
treat that null as its cue to migrate a stored `cockpit.layout` instead
of silently answering `play`. A client skipping the same step would land
every returning player in the world layout regardless of what they had
saved. Once a mode *is* set, the legacy key is not consulted — it is a
migration cue, never a live source of truth.

⚠ **`govern` falls back to `play` VISIBLY** (`ResolvedMode.fallbackFrom`).
A silent substitution makes an unbuilt mode indistinguishable from a
finished one.

**`ViewsMenu`** now enumerates `COCKPIT_MODES × COCKPIT_ARRANGEMENTS`
rather than the five legacy layout names. It was already *sending*
`cockpit mode` commands — it was building them from the wrong list, so a
mode with two arrangements could only ever be offered once. An entry the
client cannot render yet is labelled `╌╌ not built` rather than hidden:
the command works, it just lands on the floor, and hiding it would make
the cockpit's vocabulary look smaller than it is.

`noLayoutKey` (in `layouts/__tests__/modeResolution.test.ts`) keeps the
registry deleted — the key is still on the wire, and reading it is always
the shortest path back.

## Cross-references

- cockpit-composition.md — the binding layout grammar
- [client-shell.md](./client-shell.md) — the client front-door (frame primitives, clientState channel)
- [cms.md](./cms.md) — the CMS surface re-homed into `builder`
- [livestream.md](./livestream.md) — the broadcast-feed substrate (+ the overlay relay-chat envelope)
- [command-parsing.md](./command-parsing.md) / [command-routing.md](./command-routing.md) — the dispatch path the input-mode prepend hooks
- [streaming.md](./streaming.md) — the `tune`/`watch` surface driving `cockpit.watch` + the relay chat terminal shows

## History

Built in five phases (`84f79f63`→`7ca1573f`): layout axis → livestream-
viewer + embed → streamer/builder → per-bar input mode → summoned-panel
tier. A visual-review pass (`0146b7b2`) then evolved the chrome and a few
layouts past the original spec, and two follow-ons refined the input-mode
model — so the doc above describes the *shipped* shape, which diverges
from the retired requirements/plan in four deliberate ways:

- **One chrome bar, not two** — the Views switcher + Settings folded into
  `Frame`; the separate `ChromeBar` row was deleted.
- **Streamer is a control deck, not a stats placeholder** —
  `StreamerDeck` (grouped command-shortcut scaffold), chat rail widened.
- **Input modes are transient** (`6d43beb2`) — `cockpit.inputModes` is a
  `transient` clientState key (in-memory `_transientClientState`, never
  persisted, resets on fresh login); `ModeController` dropped its
  `save()`. `cockpit.layout` stays persisted.
- **`cockpit cli <id>`** (`4e82e740`) — explicit target so an un-prefixed
  affordance can scope a named bar.

The `world` layout's right rail gained the **Inspect | Who's Online**
card switch when the social-inspection feature merged from master — the
roster card is that subsystem's; the cockpit just hosts it in the rail.

## Crossing into a sandbox circle

This state **forks onto the wire body** (fork-only — a change made
inside a circle discards with it). The vessel is a baseline body, and
"baseline" means no gear and no chattel, not a stranger's defaults: a
maker who has to retype full verbs, re-pick a theme, or re-open a
layout inside their own workshop is being punished for using it. See
[sandbox.md](./sandbox.md) § the crossing.

### Wave 6 (client rebuild) — 2026-08-15

`cockpit.layout` stopped driving the client frame. `MODE_REGISTRY` is
keyed `(mode, arrangement)`; the legacy key survives on the wire and is
read in exactly one place, as the migration cue for a player with no
`cockpit.mode` — which is what the server's own `getCockpitMode()` does
with it. See § The client frame renders from both axes.
