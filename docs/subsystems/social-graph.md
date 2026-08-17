# Social-graph subsystem (Wave 3 — the attention-management layer)

The layer the [contacts](./contacts.md) bucket *storage* was always a
means to: **attention management**. In a busy social space, recognition
isn't just *who do I know* — it is *whose attention do I spend*. Two
halves, over one shared resolution primitive:

1. **Display lensing** — a per-viewer occupant block when rendering a
   room: friends boosted and full-named, strangers collapsed into
   density-aware, similarity-grouped counts. The "a 200-player tavern
   renders as a manageable scene" thesis.
2. **Notification policy** — per-*group* rules (keyed on any `GroupRef`,
   not just contacts) for whether a person's presence (entering / leaving
   the game, dropping / reconnecting) and messages surface to the player
   (`show` / `silent`), resolved by a strict ordered rule list, rate-
   limited, and delivered as an **ordinary inline message frame** (no
   separate notification surface — see "Presence frames" below).

Both halves call **one** `SocialApi.ruleFor(viewer, person)` primitive.
The store is a `NotifyPolicyMixin` per-character rule list — the
structural sibling of `_contacts`. The player surface is one dedicated
verb, `notify`, plus a thin client settings card over it.

Seeded by
social-graph-slate.md;
builds on [contacts.md](./contacts.md),
[grouping.md](./grouping.md) (the `GroupApi`/`GroupRef` policy subject),
[belief.md](./belief.md) (`RecognitionApi.describe` / `salientFeatures`),
[messaging.md](./messaging.md) (the Scene composer),
[message-rendering.md](./message-rendering.md) (the theme palette + MML),
[shell-environment.md](./shell-environment.md) (`social.verbosity`).

## Module layout

| Concept | Home |
|---|---|
| Value objects (`NotifyRule`, surfaces, `PaletteToken`, `RESERVED`) | `lib/social/NotifyRule.ts` |
| Store mixin (`NotifyPolicyMixin`, `social.verbosity` schema) | `lib/social/NotifyPolicy.ts` |
| Gated dev-facing face | `api/social.ts` (`SocialApi`) |
| Hot-reloadable logic singleton | `obj/api/SocialLogic.ts` (`/obj/api/social`) |
| The `notify` verb | `cmd/social/notify.yaml` + `obj/command/social/NotifyController.ts` |
| Presence events | `Avatar.enter` / `Avatar.onLinkdead` (+ `setLeaveIntent`), `lib/events.ts` (`PlayerLoggedIn`/`PlayerLoggedOut`/`PlayerReconnected`/`PlayerDisconnected`) |
| Country of origin | `api/connection.ts` (`ConnectionApi.originOf`/`recordOrigin`, `geoip-lite`), captured at the WS handshake |
| Client settings panel | `components/settings/SocialNotificationsPanel.tsx` (presence frames render inline — no bespoke client component) |

No new module category: `SocialApi`/`SocialLogic` mirror the
`RecognitionApi`/`RecognitionLogic` and `RenownApi`/`RenownLogic`
Api↔logic-singleton split verbatim. All real logic lives in
module-private free functions in `SocialLogic.ts` (the `RenownLogic`
discipline — public-to-public self-calls would trip the `FromModule`
gate); each public method carries
`@CallSecurity(FromModule('/api/social#SocialApi'))`.

## The rule store — `NotifyPolicyMixin`

`NotifyPolicyMixin` is composed onto `Character` at the same site as
`ContactsMixin`, so every Avatar carries it natively; NPCs opt in
explicitly. It is **pure storage** — no resolution:

```ts
export interface NotifyPolicy {
  notifyRules(): readonly NotifyRule[];          // stored order
  upsertNotifyRule(rule: NotifyRule): void;      // by groupRef, in place or append
  removeNotifyRule(groupRef: GroupRef): boolean;
  reorderNotifyRule(groupRef, anchor, where: 'above'|'below'): boolean;
}
static fieldMeta: FieldMeta = {
  _notifyRules: { persistent: true },
};
_notifyRules: NotifyRule[] = [];
```

Persisted as plain JSON via the Hydrator, the `_contacts` precedent
(legacy-tolerant `= []` default). The 50-rule soft cap is **not** here —
it's enforced at `notify` set-time, mirroring how Contacts keeps storage
dumb.

One `NotifyRule` row carries both the display treatment and the
notification surface, plus a single theme-palette `color`:

```ts
interface NotifyRule {
  groupRef: GroupRef;
  nameRendering: 'name' | 'feature-string' | 'count-only' | 'hidden';
  boostInDense: boolean;
  onConnect:    'show' | 'silent';   // arrivals: login / reconnect
  onDisconnect: 'show' | 'silent';   // departures: logout / disconnect
  onMessage:    'full' | 'summary' | 'silent';
  color: PaletteToken;        // a named token, never raw hex
}
```

The **global** `social.verbosity` dial (`minimal` | `standard` |
`verbose`, default `standard`) is the one piece that *is* a setting —
declared as `static settings` **on the mixin** (schema-on-owner, see
[shell-environment.md](./shell-environment.md)), not on the
`EnvironmentMixin` substrate. A dynamic ordered list of arbitrary refs
can't live in a settings keyspace, so the rule list is its own store.

## Policy subject — any `GroupRef`

Policy attaches to a **`GroupRef`** (grouping.md), not a bare contacts
label. The `GroupApi` facade already unifies three group kinds behind
one ref shape — **managed** (`managed:fighter-guild`), **MQL**
(`mql:…`), and **contacts** (`contacts:<pid>:<label>`) — so "notify me of
all Fighter Guild logins" is the *same* mechanism as "boost my friends."
Contacts labels are addressable by bare name at the verb boundary
(`notify friends …` → `contacts:<me>:friends`).

### The reserved baseline (virtual until edited)

Four reserved identifiers seed a sensible default order without any
stored rows:

```
foes  →  friends  →  …custom…  →  strangers  →  everyone-else
```

`foes`/`friends` normalize to `contacts:<me>:<id>`; `everyone-else` and
`strangers` are **bare pseudo-subjects** (not `GroupApi` refs).
`effectiveRuleList(viewer)` splices the player's stored rules into this
baseline: a reserved label only **materializes** a stored row once the
player edits or reorders it; otherwise it resolves at its baseline
position with baseline defaults. Baseline field defaults and a custom
rule's neutral color are read from the **`AppSettings`** keyspace
(`social.baselineRules` / `social.defaultColor`), with a code-side
literal fallback for pre-warm reads — never code constants
(the `RenownLogic.receptionWindowS` precedent). `strangers` precedes the
always-true `everyone-else` catch-all so an *unrecognized* person
resolves to `strangers` and a recognized one falls to `everyone-else`.

## The shared primitive — `ruleFor`, strict ordered first-match

```ts
SocialApi.ruleFor(viewer, person, { excludeMql? }): Promise<ResolvedRule>
```

Walks the effective list top-to-bottom and returns the **first** rule
whose group contains the person — allow or deny, full stop.
`matchesRule` dispatches: `strangers` → `!RecognitionApi.recognizes`;
`everyone-else` → always true (the tail); any other ref →
`GroupApi.isMember(personDurableId, ref)` (covers the normalized
`friends`/`foes` contacts refs, managed groups, and MQL). The person's
durable key is `playerId` for an Avatar, else `templatePath` (NPCs).

**Authority is positional** — list order *is* the precedence. A `foes`
deny placed *above* a broad allow mutes that person regardless; a
force-allow above a broad mute always fires. There is no priority
integer and no lock flag. This single walk replaces both prior conflict
rules ("max salience," "lowest priority wins").

`ruleFor` feeds **two** consumers — the display formatter and the login
fan-out — and neither reimplements it. That "one resolution, two
consumers" contract is the spine of the build.

## Display lensing — the per-viewer occupant block

`SocialApi.composeOccupants(viewer, occupants, roomSize): Promise<Mml>`
is the formatter — a sibling of `RecognitionApi.describe` *one
cardinality up*: `describe` names one target viewer-aware; the formatter
orders / groups / collapses a *collection* and composes **through**
`describe` per named occupant (never re-implements naming).
`LookController.lookAtLocation` calls it at the single existing
`Mml.list(...)` occupant seam, so both `look` and arrival (which reuses
`look` via `forceCommand`) route through one chokepoint; non-organism
loose contents stay on the plain `ContainmentApi.looseContents` path.

### Density tiers + verbosity

`roomSize` (renderable-organism count) drives the four-tier table; the
viewer's `social.verbosity` modulates it (`minimal` shifts one tier more
aggressive, `verbose` disables collapse via a `-1` level):

- **`<10`** — every occupant rendered fully.
- **`10–30`** — `everyone-else` named; `strangers` as feature-strings.
- **`30–100`** — `strangers` aggregated to similarity-grouped counts;
  `everyone-else` named; `boostInDense` rules lifted above.
- **`>100`** — only boosted rules named; the rest as grouped counts /
  `(N others present)`.

A `boostInDense` occupant is always lifted and full-named (carrying its
rule's `color` as an MML `<name … color="amber">` attribute the client
theme maps via the stylesheet cascade); a `hidden`-render rule is
dropped.

### Similarity grouping

Collapsible occupants group by the tuple **(species, most-distinctive
worn feature)** — species via `OrganismMixin.getSpecies`, the worn
feature parsed out of `RecognitionApi.salientFeatures` (compose *through*
the shipped primitive, never re-derive). A group needs a shared tuple
**and** ≥2 members to read as a count line ("12 dwarves in red robes");
lone occupants and incomplete tuples fall to the generic "(N others
present)" bucket.

### The aggregate stays targetable

A collapsed line is **not** a dead string: it's a `mudq:` MML `<link>`
handle carrying its room-scope MQL seed (`dwarves in red robes` /
`others`). **What v1 ships is the painted seed only** — the seed is
present and tested, but the four richer client interactions it's meant
to feed ([client-shell.md](./client-shell.md) hover→command-bar
preview, expand-on-pull, the [inspection-card](./card-surface.md)
drill-in roster, and the [prompt.md](./prompt.md) `mqlMany` verb-time
pick-list) are **not yet wired client-side** (`mudq:` renders as an
inert painted span); that wiring is a deferred seam. The principle
still holds — collapse is a display lens, never a targeting wall:
ordinals / feature-filters / post-`look` pronoun memory already resolve
the collapsed occupants against the live room scope by *typing*, with
or without the click affordances. (The seed is also not yet
percent-encoded — raw spaces — which lands with the click handler.)

Cost is bounded — display resolves `ruleFor` once per *visible occupant
in one room* (room-size bounded), once per render; MQL refs are valid
display subjects, evaluated once per render.

## Notification policy — the presence fan-out

`Events.PlayerLoggedIn` / `PlayerLoggedOut` were emitted but had **no
buffer-relay consumer**. This build supplies one, and refines the event
taxonomy into **four player-level transitions, each gated on having a
character in the world** (a bare welcome-screen socket and the
OAuth/user layer never surface):

| Event | Fires from | Meaning |
|---|---|---|
| `PlayerLoggedIn` | `Avatar.enter` (fresh instance) | a character entered the game |
| `PlayerReconnected` | `Avatar.enter` (instance survived linkdead) | a connection returned to a still-in-world body |
| `PlayerDisconnected` | `Avatar.onLinkdead` (no leave intent) | the last socket dropped involuntarily (linkdead); the body lingers |
| `PlayerLoggedOut` | `Avatar.onLinkdead` (leave intent set) | a deliberate sign-out / switch-character |

The fresh-vs-reconnect split keys on a transient per-instance
`sessionActive` flag (set in `enter`, surviving the linkdead window — a
real logout destructs the instance, so the next session starts fresh).
The drop-vs-logout split keys on a transient `leaveIntent` flag: the
client closes the socket with the **`INTENTIONAL_LEAVE_CLOSE_CODE`**
(4000) on a deliberate teardown, threaded `Backend.handleWebSocketClose`
→ `Application.handleUserDisconnect(socketId, intentional)` →
`Avatar.setLeaveIntent` before the linkdead edge fires. A raw drop leaves
the flag unset. The relay's only non-test consumer of these events is
itself, so the taxonomy is free to be precise.

`SocialApi.boot()` (wired from `AppBootstrap.run()` after
`RenownApi.boot()`) installs an idempotent presence tap on `SocialLogic`
(the `RenownLogic` tap shape) subscribing to all four. `relayPresence`:

1. Resolves the acting Avatar; scans every online viewer
   (`PlayerApi.getAllAvatars()`, skipping the actor and any destroyed /
   linkdead handle).
2. `ruleFor(viewer, actor, { excludeMql: true })` — first match.
3. Arrivals (`loggedIn` / `reconnected`) read `rule.onConnect`;
   departures (`loggedOut` / `disconnected`) read `rule.onDisconnect`.
   `silent` → skip. The two policy directions cover all four events; the
   rendered line names the specific transition.
4. **Rate-limit** per `(actor, viewer, event)` via an in-memory
   `Map<string, number>` window (60 s; transient, nothing persisted —
   the `RenownLogic.receptionSeen` precedent; cadence is mechanism, so a
   code constant rather than an AppSettings dial). A flapping connection
   is dropped within the window.
5. Sends a `session.presence` frame:
   `MessageApi.scene(viewer).topic(...).toSelf(body, payload).send()` —
   the line viewer-aware (`Mml.actor`), tinted **inline** by the rule
   `color` (a `<highlight>` wrap, mirroring `styleMessageForImpl`).
   Arrivals append "from `<country>`" when the origin resolved (below).

This is **global presence** — surfaced wherever the viewer is — and is
the only substantive new wiring. **Room movement is not a notification
event**: enter/leave stays the existing ungated movement line, with no
per-bucket knob.

### Customizing the line — `social.presenceFormat`

The presence line is **author-owned**: a per-character **Liquid template**
held in the `social.presenceFormat` setting (schema-on-mixin on
`NotifyPolicyMixin`, so it persists on the avatar like any setting). The
relay renders it per-viewer through `ProseApi.format` (Mml-aware, so the
late-bound `who` ref stays viewer-aware and clickable), then tints the
result inline by the matched rule's `color`. A syntactically broken
template never silences the notice — it falls back to the shipped default.

The variable context (the documented extension seam — `PRESENCE_VARS` in
`NotifyRule.ts`) is where future fields graft on (player level, guild,
pronouns, time-of-day) with no plumbing change:

| Variable | Meaning |
|---|---|
| `who` | the actor, viewer-aware (recognition-gated, clickable) |
| `action` | the default phrase ("entered the game" / "reconnected" / …) |
| `event` | the raw key (`loggedIn` / `reconnected` / `loggedOut` / `disconnected`) |
| `category` | `arrival` or `departure` |
| `is_arrival` | boolean |
| `country` | country of origin (arrivals, when resolved), else null |

Default: `{{ who }} has {{ action }}{% if country %} from {{ country }}{% endif %}.`
Set it via `settings set social.presenceFormat "<template>"` (quote it so
the Liquid `{{ … }}` survives the command tokenizer — `settings set`
strips the wrapping quotes on store) or the Social card's "presence line"
field. The pushed `social.rules` projection carries the current value so
the card shows + edits it; writes route back through the same `settings`
command (the buttons-preview-their-command contract).

### Bounds + privacy

Cost is `O(online viewers × rules × isMember)`. It's bounded because (a)
the default surface for non-contacts is `silent` (only bucketed
non-silent rules emit), and (b) **MQL refs are excluded as notification
subjects** (`excludeMql` skips `parseRef.source === 'mql'`) — a live
membership query per login is the expensive case; MQL refs stay valid
*display* subjects. No persisted reverse index; a persisted reverse index
is the deferred Layer-2 if `getAllAvatars()` grows large.

The fan-out runs as **substrate code** (no command context): the match
comes from the *viewer's own* rule list under the viewer's identity, and
the pushed frame carries only the actor ref / surface / color — never the
rule list or who-policied-whom. This matches the owner-only
`ContactsGroupProvider` boundary; policying a group never leaks the
viewer's rules to its members or to third parties.

### The MQL display-vs-notification split

| Ref kind | Display subject | Notification subject |
|---|---|---|
| `contacts:` / `managed:` | yes | yes (cheap roster lookup) |
| `mql:` | yes (once per render) | **no** (excluded — too costly per login) |

## The `notify` verb

The attention-rule list is owned by a **new dedicated verb** — *not*
`contacts` (your private lists) and *not* `group` (managed-group admin).
A `notify` rule is your **private lens over any group**, owner-private
like contacts but with an arbitrary `GroupRef` subject.

```
notify                          # list your ordered rules + a compact summary
notify <ref>                    # show one rule
notify <ref> --login <s> --disconnect <s> --message <s> \
       --render <s> --boost --no-boost --color <s>
                                # set fields (typed options)
notify <ref> --above <ref>      # reorder above another (order = precedence)
notify <ref> --below <ref>      # reorder below another
notify <ref> --remove           # drop the rule (group falls to the tail)
notify remove <ref>             # …same, via the remove subcommand
```

The set-fields are typed **options**, not positional `k=v`: the command
framework forbids an optional `<ref>` positional followed by a greedy
assignment positional (a greedy arg is implicitly required + last), so an
optional ref + free-form `k=v` is structurally illegal.

`NotifyController` is a thin caller: it normalizes the typed `<ref>`
(bare label → `contacts:<me>:<label>`; refs containing `:` and the bare
pseudo-subjects pass through), builds the field patch from whichever typed
options were provided (validating each value against its vocabulary).
`--login` / `--disconnect` take `show|silent` (a presence frame in the
buffer, or nothing — `--login` covers arrivals, `--disconnect` covers
departures); `--message` takes `full|summary|silent`; `--render` takes
`name|feature-string|count-only|hidden`; `--boost` / `--no-boost` are
the boolean flag pair; `--color` takes a palette token. It then enforces
the **50-rule soft cap** at set-time with a friendly rejection, and
dispatches
to `SocialApi.{setRule,removeRule,reorderRule,listRules}`. A `silent`
surface *is* the mute, so allow and deny share the verb. The global
verbosity dial stays the settings verb
(`settings set social.verbosity standard|minimal|verbose`).

## Client surfaces

### Presence frames render inline (no separate surface)

A presence frame rides the ordinary `MessageFrame` channel (no new wire
type) on `topic === 'session.presence'` and renders **inline in the
message buffer like any other scene frame** — there is no toast / queue /
overlay. (An early iteration routed a `banner` surface into a dismissable
`NotificationQueue.tsx` toast stack; that was deliberately removed —
presence is "a simple message frame," not a new client notification
category. The client just lets the frame fall through to the normal
per-topic / catch-all append.) The line is tinted **server-side** by the
rule `color` (a `<highlight>` wrap the `MmlRenderer` resolves through
`tokens.palette` — a named token, not hex, the same palette the boosted
room name and the message highlight use, so a theme swap re-tints every
social highlight in one edit). The structured `SocialNotificationPayload`
(`{kind, event, actor, color, country?}`) rides along for any future
structured consumer.

### The settings card

`SocialNotificationsPanel.tsx` ("Social / Notifications", reachable from
the `AccountMenu`, toggled via a `socialPanelOpen` store flag) is a **thin
front over `notify`**:

- **Reads** the server-pushed `clientState['social.rules']` projection
  (`SocialRulesState` in `@saxonberg/types`) reactively. After every
  mutation — and on a bare `notify` list — `NotifyController` calls
  `host.pushClientStateUpdate('social.rules', projection)` (the
  style-overlay precedent; skipped for an NPC host with no connected
  Interactive). `social.rules` is **not** a persisted client-state key —
  a pure push cache. The rule store stays the single source of truth; the
  card never writes it directly. On mount with no cached projection the
  card issues a bare `notify` to request one.
- **Every control previews its command** (the global "buttons preview
  their command in the command bar" contract): hovering a control
  previews the equivalent `notify …` via `onCommandPreview`, and
  committing issues it via `onCommandClick` — the same path the command
  bar uses. Controls are option buttons + color swatches (one command per
  button), so the contract is exact and testable.
- **Reorder is up/down buttons** (order = precedence): "up" issues
  `notify <ref> --above <prev>`, "down" issues `notify <ref> --below
  <next>`. Drag-to-reorder is the ideal, but fiddly in this stack and an
  extra dependency; the up/down buttons issue the same `--above`/`--below`
  and the server's ordered store is authoritative, so the projection's
  next push reconciles the order.
- An "add group" input resolves a typed ref to a default rule
  (`notify <ref> --render name`, a default-preserving create), and a global
  `social.verbosity` control issues the `settings set` command.

## Country of origin

Arrivals (`loggedIn` / `reconnected`) gain a "from `<country>`" tail when
the connecting player's country resolves. The capture is a v1 slice of
the connection-origin slate:

- **Capture** at the WS handshake (`WebSocketService.handleUpgrade`):
  the client IP (first hop of `X-Forwarded-For`, else
  `socket.remoteAddress`) is threaded `Backend.handleWebSocketConnect` →
  `Application.handleUserConnect` → `ConnectionApi.recordOrigin`, which
  derives the country via the offline **`geoip-lite`** dataset
  (`Intl.DisplayNames` for the ISO→name render) and stashes
  `{ ip, country }` **transiently on the `Interactive`** (in-memory only;
  never persisted — the PII posture).
- **Expose**, privilege-split: `ConnectionApi.originOf(playerId)` returns
  **country only** (broadly readable); the raw IP never leaves the
  connection layer (the developer-gated IP read stays deferred).
- The relay reads `originOf(actor).country` for arrivals only (a
  departure has no origin). It resolves to nothing on **localhost /
  private IPs** — so the country line only appears against real remote
  connections. For local testing, set the **`DEV_GEO_COUNTRY`** env var
  (e.g. `DEV_GEO_COUNTRY=Germany`): an unresolved IP then reads as that
  country. Dev-only — production has real client IPs.

City/region, the developer-gated IP read, and any persisted "last-seen
country" remain deferred to the slate.

## A flagged deferral

1. **Message-restyle live wiring (Phase 3b).**
   `SocialApi.styleMessageFor(viewer, speaker, body)` is implemented and
   unit-tested — it applies the first-match `onMessage` surface (`full` /
   `summary` → highlight in the rule color, `silent` → notification
   suppression only, **never** feed-filtering) and is late-bound
   per-recipient. But it is **not yet consulted by the live message
   path**. The clean per-recipient seam (`SensorMixin.onMessage` is a
   framework template method; the speech producers compose one
   multi-recipient `Scene`) would need a sync contacts-fast-path to clear
   the multi-recipient `async ruleFor` wall — deferred rather than forced
   here. `summary` currently renders like `full` (no clean per-recipient
   aggregation hook yet). A future `filterMessage`-shadow or late-bound
   producer-side wrapper calls `styleMessageFor` at the compose seam.

## Non-goals (this build)

- **Message filtering / moderation** — a `foes` policy governs display
  de-emphasis + notification suppression only; dropping speech from a
  feed is a comms concern (comms-slate).
- **Mutual / consent friending** — bucketing stays unilateral + private.
- **`onProximity` / `onActivity` / movement notifications** — v1 covers
  presence (login / logout / reconnect / disconnect) + message.
- **MQL refs as notification subjects** — display only.
- **Raw / custom highlight colors** — named theme-palette tokens only.
- **Account-level bucket federation across characters** — per-character
  v1 (Wave 4).

## Related

- [contacts.md](./contacts.md) — the bucket storage substrate
  (`ContactsMixin`, `_contacts`) the rule store is a sibling of; the
  `contacts:<pid>:<label>` ref the reserved `friends`/`foes` normalize to.
- [grouping.md](./grouping.md) — the `GroupApi` facade + `GroupRef` shape
  (the policy subject) + the owner-only `ContactsGroupProvider` boundary.
- [belief.md](./belief.md) — `RecognitionApi.describe` / `salientFeatures`
  (the compose-through primitives).
- [messaging.md](./messaging.md) — the Scene composer + sensor routing the
  presence frame rides; [message-rendering.md](./message-rendering.md) —
  the theme palette + MML the `color` token resolves through.
- [shell-environment.md](./shell-environment.md) — the `social.verbosity`
  settings keyspace; [app-settings.md](./app-settings.md) — the
  deployment-default baseline-rule seeds.
- [client-shell.md](./client-shell.md) — the frame / store pattern
  presence frames render through + the command-bar preview
  contract; [card-surface.md](./card-surface.md) — the drill-in
  roster; [prompt.md](./prompt.md) — `mqlMany` verb-time disambiguation.
- social-graph-slate.md — the
  seeding slate (Wave 3 shipped here).

## History

Built as social-graph Wave 3 (commits `d84b0688`..`123f5172`, merged
2026-06). Three design→implementation shifts worth recording, since the
plan and requirements (now retired) described the earlier shape:

- **Display formatter resolves eagerly per-viewer, not late-bound.** The
  plan modeled the occupant block as a synchronous late-bound
  `{ toMml(viewer) }` fragment. `ruleFor` is **async** (membership rides
  `GroupApi.isMember`) and MML `toString(viewer)` is synchronous, so the
  formatter instead resolves eagerly for the single known viewer at the
  `look`/arrival seam (both are single-recipient). Late-binding for a
  future multi-recipient enter-broadcast is deferred.
- **`notify` set-fields are typed `--options`, not positional `k=v`.** A
  greedy positional is implicitly required and must be last, so it can't
  follow the optional `<ref>` — the `k=v` grammar the plan showed could
  not load. See the *The `notify` verb* section.
- **Message restyle is method-complete but unwired (Phase 3b).**
  `styleMessageFor` exists and is tested; wiring it into the
  multi-recipient speech path hits the same async-`ruleFor`/sync-render
  wall and wants a sync contacts-fast-path — deferred.

## ⭐ Idleness — the project standard

> **You are idle when you have not sent a command.** Not when the socket
> is quiet, not when a heartbeat lapses, not when a widget says so.

`Interactive.lastInputAt` is stamped by `touchInput()` at the
**`CommandGiver` dispatch tail** — so it advances on every dispatched
command and on nothing else. Because every clickable in the cockpit
sends a command, **clicks count automatically**; there is no separate
click signal to keep in step, and there must never be one.

`PresenceLogic` derives the status by comparing that stamp against the
configurable `social.idleAfter`. There is **no stored idle flag and no
per-player timer** — idleness is derive-on-read like every other
standing in this codebase.

⚠ **Do not introduce a second clock.** Anything that needs "is this
player paying attention" — presence rows, the notification widget,
idle-eviction, away summaries — reads this one. A subsystem that starts
tracking its own activity signal will disagree with `who` the first
time the two are computed a second apart, and the disagreement will be
invisible.

### ⚠ Attention is idleness; a cockpit mode is only *where* you are looking

The cockpit's mode axis (`chat · play · watch · build · govern`) is a
declared intent, not a measurement: a player can sit in `play` while
away from the keyboard, or work in `build` while glancing at the
terminal rail constantly. So the division of labour is fixed:

| Question | Answered by |
|---|---|
| Is this player paying attention at all? | `lastInputAt` — **the truth** |
| *Which surface* counts as the one they are watching? | the cockpit mode |

A consumer that used mode as a proxy for attention would mute a player
who is actively working and alert one who has walked away.

## Roster rows across the sandbox boundary

`SocialApi.statusOf` and `SocialApi.composeRow` route through
`SecurityApi.projectAcross` (see
[call-security.md](./call-security.md)). A roster row IS the per-viewer
projection of a person — the same category as naming — and `who` from
inside a circle composes one for every online person, all of them
field-resident. Un-apertured the verb simply threw.

The presence relay resolves its delivery target through
`SandboxApi.activeBodyFor`: rules stay on the registry avatar, delivery
follows the live body. See [sandbox.md](./sandbox.md).
