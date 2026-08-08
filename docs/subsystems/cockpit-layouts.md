# Cockpit layouts + server-authoritative input mode

The client front-of-house arrangement layer. Two server-authoritative
axes ride the existing `clientState` channel — **`cockpit.layout`** (what
the cockpit looks like) and **`cockpit.inputModes`** (how typing is
scoped, per bar) — and the client is a pure view over them. The governing
principle: **the client owns zero command semantics.** Every layout
switch and every mode change is a real command on the wire.

Built against the binding composition grammar
(never-blind, fixed-chrome/fluid-content, hierarchy-encoding splits, the
no-modal rule).

## The layout axis

Layout is per-player UI state, identical in kind to `console.tabs` — so
it is a `clientState` key, not a client-local field.

- **Vocabulary** — `LayoutName` / `LAYOUT_NAMES` in `@saxonberg/types`,
  imported by both ends so the verb's validator and the client registry
  can never drift: `world` | `forum` | `livestream-viewer` | `streamer`
  | `builder`.
- **Schema** — `cockpit.layout` is a `clientStateSchema` entry on
  `HasInteractiveMixin` (`mud/lib/connection/HasInteractive.ts`), default
  `'world'`, validator = `LAYOUT_NAMES.includes(v)`.
- **Verb** — `layout <name>` (`cmd/shell/layout.yaml` +
  `obj/command/shell/LayoutController.ts`), mirroring `StyleController`'s
  **write → save → push** commit triple (`setClientState` →
  `Avatar.save()` → `pushClientStateUpdate`). Unknown names reject via a
  `controller-rejected` note; the write never lands. Co-located with
  `style` in the cockpit-config verb family (`style` = appearance,
  `layout` = arrangement, `mode` = input scope).
- **No auto-switch.** `layout` is the *only* way to change layout — typed
  or sent by the "Views" menu. The `forum` verb stays pure CRUD (you reach
  the board view via `layout forum`); the standalone `?surface=cms`
  takeover is retired (builder is entered the same way). No domain verb,
  URL, NPC, or item flips layout this cycle (deferred).

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
  preview/flash live in the ghost line, not in a bar.
- The **"Views" menu** (`components/ViewsMenu.tsx`) previews/sends
  `layout <name>` per the click model and marks the current layout. It +
  the Settings affordance live in the **single top `Frame` bar**
  (`components/frame/Frame.tsx`) alongside bus-health + identity — one
  fixed chrome row, not two (the separate `ChromeBar` row was folded into
  `Frame` so the content area reclaims the vertical space).
- An **always-on minimum** (the Frame bar, a command bar) renders in
  every layout, so the player can always type `layout world` to leave.

### The five layouts

| Layout | Shape (canonical split) | Notes |
|---|---|---|
| `world` | terminal + a right rail with an **Inspect \| Who's Online** pane switch (Single + rail) | the classic cockpit; the rail pane is chosen by the `rightPane` store slice (inspection detail vs the social-inspection roster) |
| `forum` | board view + chat sidecar | the old `mainView === 'forum'`; live-scene peek keeps it never-blind |
| `livestream-viewer` | video embed (focal) + game terminal + chat rail | see below |
| `streamer` | a **control deck** (focal) + game terminal + a widened chat rail | `StreamerDeck`; see below |
| `builder` | CMS editor (dominant) + glance terminal rail (Monitor) | the CMS re-homed in-session |

`LivestreamViewerLayout` and `StreamerLayout` share `LivestreamPanes`
(the game terminal + chat rail + bar); only the focal pane differs (the
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
run in the ghost line on hover (the same learn-the-CLI affordance) but is
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
sends `layout builder` over the bus instead of opening a tab.

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
  bar's entry — `mode <prefix…>` sets, `mode off` / bare `mode` clears.
  The target resolves `model.bar ?? context.barId ?? 'main'`: a typed
  `mode` defaults to the bar it was submitted from (`context.barId`), and
  the explicit **`--bar <id>`** option names a bar for an affordance —
  which dispatches *un-moded* (no wire `barId`, so preview == send) and so
  can't rely on `context.barId`. The target rides in the command text
  (`mode chat --bar stream-chat`), keeping the ghost-line preview honest.
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
  `mode off` from that bar. The chat sidecar's "talk here" sends
  `mode chat <handle>` from the forum bar.

### The ghost command line + click model

Under per-bar mode, preview cannot live *in* a bar (a moded bar would
prepend its prefix, so the preview would lie). It lives in the always-on
**ghost command line** (`components/GhostCommandLine.tsx`, store-backed
`ghostPreview`/`ghostFlash`) beside the primary bar. Hover → the exact
command; **click → run it un-moded** (no `barId`, so preview equals
send); **shift-click / right-click → copy** to the clipboard (the
explicit "make it mine" path, replacing shift-click-loads-a-bar, which
has no honest target under N bars).

## The summoned-pane tier

The no-modal rule (composition law 6) routes would-be modals into two
tiers that both keep a terminal on screen: the **layout tier** (above)
and the **summoned-pane tier** — transient panes that dock *beside* the
current layout's terminal (the inspection pane is the existing proof).
This build establishes the summoned-pane tier as a first-class mechanism:
a `summonedPane` store slice (`open`/`closePane`) and an `App`-level
`ContentRow` that docks the pane beside the active layout.

**Settings** is the first consumer (`components/settings/SettingsPane.tsx`):
a non-modal side panel whose controls each **send the real command**
(`settings set …` / `var set …`, un-moded) per command-bus primacy — no
client-only settings state. Its notification section consumes the
(separately-built) notification-settings backend when present and
**degrades to a "coming soon" placeholder** when absent (the case on this
branch). Opening the pane is client UI state (like the inspection pane),
not a `layout` switch — it coexists with the current layout.

## Cross-references

- cockpit-composition.md — the binding layout grammar
- [client-shell.md](./client-shell.md) — the client front-door (frame primitives, clientState channel)
- [cms.md](./cms.md) — the CMS surface re-homed into `builder`
- [livestream.md](./livestream.md) — the broadcast-feed substrate (+ the overlay relay-chat envelope)
- [command-parsing.md](./command-parsing.md) / [command-routing.md](./command-routing.md) — the dispatch path the input-mode prepend hooks
- [streaming.md](./streaming.md) — the `tune`/`watch` surface driving `cockpit.watch` + the relay chat terminal shows

## History

Built in five phases (`84f79f63`→`7ca1573f`): layout axis → livestream-
viewer + embed → streamer/builder → per-bar input mode → summoned-pane
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
- **`mode --bar <id>`** (`4e82e740`) — explicit target so an un-moded
  affordance can scope a named bar.

The `world` layout's right rail gained the **Inspect | Who's Online**
pane switch when the social-inspection feature merged from master — the
roster pane is that subsystem's; the cockpit just hosts it in the rail.

## Crossing into a sandbox circle

This state **forks onto the wire body** (fork-only — a change made
inside a circle discards with it). The vessel is a baseline body, and
"baseline" means no gear and no chattel, not a stranger's defaults: a
maker who has to retype full verbs, re-pick a theme, or re-open a
layout inside their own workshop is being punished for using it. See
[sandbox.md](./sandbox.md) § the crossing.
