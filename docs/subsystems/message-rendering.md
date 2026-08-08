# Message Rendering

How a `MessageFrame` becomes pixels on the cockpit. The substrate spans
both sides: the **server** stamps a tagged-complete-string body in MML;
the **client** parses it into a tree, picks a per-message-type
template, and paints through a reader-owned stylesheet.

The load-bearing rule from
message-rendering-slate.md:

> **The message string captures everything — always.** A frame's `body`
> is the complete canonical line (channel chip + sender + content +
> everything). **Flatten (strip tags) = the failsafe.** The rich client
> reflows from the *same* tags into columns / hanging indent / etc.
> Same artifact, viewed two ways.

Wave 1 ships the model end-to-end. Wave 2 (layout library — `<box>` /
`<table>` / `<list>` rich render) and Wave 3 (channel stylesheets —
community-authored custom tags + palette) are deferred to the slate.

## Server-side composition

### MML vocabulary extensions

`Mml` (`mud/api/mml.ts`) grows the semantic core. Each new tag has a
defined flatten so the failsafe string remains complete and readable.

**Identity / role:**

| Tag | Helper | Flatten | Role |
|---|---|---|---|
| `<chan id="…">[Label]</chan>` | `Mml.chan(id, label)` | `[Label]` | Channel chip. |
| `<msg>…</msg>` | `Mml.msg(body)` | children verbatim | User-content region of chat/say/tell/emote. |
| `<player stuff-id="…">…</player>` | `Mml.player(stuff)` | display name | Player-identity sibling to `<name>`. |
| `<npc stuff-id="…">…</npc>` | `Mml.npc(stuff)` | display name | NPC-identity sibling to `<name>`. |
| `<mention stuff-id="…">@Name</mention>` | `Mml.mention(stuffId, label)` | `@Name` | Resolved explicit mention. |
| `<link href="…">…</link>` | `Mml.link(href, label)` | label | In-world ref (scheme-routed). |

**Inline emphasis (markdown subset):**

| Tag | Helper | Flatten |
|---|---|---|
| `<strong>` | `Mml.strong(body)` | `**…**` |
| `<em>` | `Mml.em(body)` | `*…*` |
| `<code>` | `Mml.code(text)` | `` `…` `` |
| `<pre>` | `Mml.pre(text)` | ` ```…``` ` |
| `<blockquote>` | `Mml.blockquote(body)` | `> …` (per line) |
| `<strike>` | `Mml.strike(body)` | `~~…~~` |
| `<list>` / `<list ordered="true">` | `Mml.unorderedList(items)` / `Mml.orderedList(items)` | linear-labeled lines |
| `<li>` | `Mml.li(body)` | `- ` / `N. ` prefix |

**Color is the one presentational tag.** Alignment and size stay
stylesheet rules, never authored prose — but a thing's *color* is a
real property of what's perceived (visible light has color), so MML
names it explicitly: `<color value="…">…</color>`
(`Mml.color(value, body)`; flatten: children verbatim). The value is a
theme-palette token or a friendly color name (`purple`/`blue`/`red`/
`grey`, aliased to the palette tokens client-side in `paletteFor`), so
it resolves through the same theme cascade as every other color and
stays legible under any theme — **never a raw hex**. The renderer tints
a `ColorSpan`; when `<color>` wraps a clickable identity tag the inner
span's color wins, so the affordance stays clickable *and* takes the
color (the TPA terminal's status-light name tint is the first consumer
— see [fasttravel.md](./fasttravel.md)). Alongside it,
`Mml.text(value)` is the escaped-text fragment: it lifts a raw string
into an `Mml`, escaped once at construction, so any display value can
join the compose chain as a fragment and is never re-escaped
downstream.

The `<sys>` tag was **retired** in an earlier build — chrome-label
sites (the `── ` glyph in `look`'s exits and contents lines) emit the
literal text as plain prose; Wave 2's structured layout vocabulary
(`<box title>` / `<list>` with header) will replace them when it ships.

### `Mml.flatten` vs `Mml.stripTags`

Two text-projection paths sit on the same parse machinery:

- **`Mml.stripTags(body)`** drops every tag, decodes the five entities,
  returns the bare plain string. Used by the `style plain on` v1
  collapse — acceptance criterion #23 wants "no italic, no bold; only
  the linear string."
- **`Mml.flatten(body)`** emits each tag's per-tag failsafe form
  (markdown emphasis preserved, lists/quotes serialized linear-labeled).
  Round-trips with `markdownToMml` for log capture, archive exports,
  and the markdown round-trip tests.

The flatten table lives in `api/mml/flatten.ts`; consumers shouldn't
import it directly — see "Module isolation rule" below.

### `Mml.markdownToMml` — Discord-dialect subset

Server-authoritative markdown parsing. Clients never re-parse markdown;
what reaches the client is canonical MML.

Recognised forms (from `api/mml/markdown.ts`):

- `**bold**`, `*italic*` / `_italic_`, `` `code` ``, ` ```block``` `,
  `> quote`, `~~strike~~`
- `- ` / `1. ` at line-start for unordered/ordered lists (one level)
- `label` for in-world refs (see "Custom URI schemes")
- `@<word>` for mentions (resolved against the call site's resolver)

**Out of scope:** GFM tables (deferred to Wave 2 layout contexts), nested
lists, arbitrary HTML, markdown headers.

### Mention resolvers

`Mml.markdownToMml(text, resolver)` accepts a `MentionResolver`:

```typescript
interface MentionResolver {
  resolveMention(word: string): string | null;  // stuffId, or null on miss
}
```

Two factories are provided:

- **`Mml.perceiverMentionResolver(speaker)`** — matches against the
  speaker's perceivable neighborhood (same set MQL's `reachable` scope
  walks). Used by `say` / `tell` / (future) `emote`.
- **`Mml.channelMentionResolver(participants)`** — matches against an
  explicit participant set. The future chat-channel substrate will
  construct the iterable at emit time; the test suite uses it with a
  fixture set.

**Silent on miss** is the contract — an unresolved `@word` stays as
plain text. No error frame; the parser leaves the literal text in
place.

### Custom URI schemes

`label` markdown produces `<link href="URI">label</link>` only
when the URI uses one of three project-owned opaque schemes:

| Scheme | Payload | Click effect |
|---|---|---|
| `mudcmd:` | percent-encoded command line | Dispatch the decoded line through the command bus *as the clicker*. |
| `mudref:` | stuff-id | Registry hit → `look <primaryKeyword>`; miss → `look #<id>`. |
| `mudq:` | percent-encoded MQL | **Inert in v1.** Namespace reserved; renderer paints styled-but-non-clickable text. |

`http(s)://` is namespace-reserved for future channel-stylesheet /
system-message contexts but **stripped from chat in v1**. Any other
scheme is also stripped at parse time (default-deny). The label
survives as plain text in all strip cases.

The custom schemes are opaque per RFC 3986 (`scheme:payload`, no `//`),
matching the convention used by `mailto:` / `tel:` / `urn:` / `data:`.
`Mml.link` validates `href` against the allowed prefix list at compose
time; user-input link parsing (which is where unknown schemes get
stripped) happens in the markdown parser, not in `Mml.link`.

The click dispatches as the **clicker**, not the sender. A malicious
`[click](mudcmd:rm /home)` does whatever the clicker would do typing
the command — sender access does not leak. The hover-preview
plumbing (`onCommandPreview`) shows the resolved command in the input
before the click commits, so click-jacking is visible.

### Module isolation rule

`Mml`'s implementation is split into a sibling `api/mml/` directory
(parser tree, flatten, markdown, mention, schemes, entities). The
directory is **private to `api/mml.ts`** — outside callers must
import from `'../api/mml'` and use the `Mml` surface; nothing else
may reach into `api/mml/`. Same convention as `api/mql/`. Code
review gates this: grep `from '.*api/mml/'` to audit.

If a consumer needs something currently only exposed in the subdir
(a type, a helper), re-export it from `api/mml.ts` — don't reach in.

### Topic literals + TopicCatalogue

The `MessageApi.Topics` constant tree was **retired**. Topic strings
are emitted as dotted-path literals at call sites:

```typescript
MessageApi.scene(speaker)
  .topic('speech.vocal')       // literal, not MessageApi.Topics.X
  …
```

The authored source of truth for the topic vocabulary lives on
per-topic YAML leaf Ideas under `seeds/lib/messaging/Topic/`. The
`TopicCatalogue` singleton self-loads them at boot and the snapshot
ships to clients in the welcome payload. See
[topics.md](./topics.md). Keeping a parallel constant tree in code
was duplication; literals + the YAML catalogue is one source of truth.

`MessageApi.Tags` stays as constants — audience tags are framework
metadata, not authored content.

## Capability mixins for outgoing comms

### VocalMixin — acoustic transport

Owns `say` (and future `whisper` / `shout` / `sing`). Composes onto
hosts whose body can produce sound. See
[messaging.md § VocalMixin](./messaging.md#vocalmixin-and-the-containable-wins-rule).
The user-supplied text runs through `Mml.markdownToMml` with the
speaker's perceiver mention resolver before being wrapped in
`<speech>`.

### AetherMixin — non-acoustic transport

Owns `tell` (and future channel `chat` / remote `emote`). The Aether
is the in-fiction comm-network — diegetically reflavored per campus
zone (implant network, magical resonance field, hybrid mesh). The
mixin codifies "this Stuff transmits and receives over the Aether";
content owns the campus-specific narrative.

```typescript
const AvatarBase = PostRegistrationMixin(
  HasInteractiveMixin(AetherMixin(ContactsMixin(ShelledCharacter))),
);
```

Avatars compose `AetherMixin` because players have implants (per the
char-gen / augmentation slates' diegetic story). NPCs opt in
per-class when content requires it.

`AetherMixin.tell(target, text)` fires a scene at `speech.comms`
with **chat-form** bodies:

```
<speaker> → <target>: <body>           ← actor frame
<speaker> → you: <body>                ← target frame
```

Self IS a valid target — note-to-self is a real use case, and the
MQL scope (`online`) + `mustBeAgent` validator leaves the speaker in
the candidate pool.

**Future composition gate (emotes slate):** SoulMixin alone grants
**local** emote (in-room, no network). The **remote** emote requires
BOTH `SoulMixin` AND `AetherMixin` — the performance is generated by
the soul, the carrier signal across rooms is the Aether. The emotes
build should honor this gate.

### Why two mixins instead of one

A character can be Vocal without being Aetheric (mute implanted /
pre-augmentation) and Aetheric without being Vocal (post-vocal-loss
with an implant). Both compose independently onto the host. The
diegetic story is set-piece for content; the mechanical separation is
permanent.

### Verb-controller shim pattern

Controllers stay thin shims that narrow the speaker to the capability
mixin and delegate. `SayController` for `say`, `DmController` for
`dm` (alias `tell`). Each:

1. Checks `MixinApi.isVocal(speaker)` / `MixinApi.isAether(speaker)` —
   on miss emits a polite refusal scene + a `mixin-missing` note.
2. Calls `speaker.say(text)` / `speaker.tell(target, text)`.

All composition (chat-form bodies, markdown parsing, mention
resolution, audience routing, payload) lives on the mixin. No
markdown handling reaches the controller layer. The pattern is
documented in [command-spec.md](./command-spec.md).

## Client-side rendering

### `parseMml` — nested-aware MML parser

`packages/client/src/lib/mml/parseMml.ts` is a state-machine parser
that returns an `MmlNode` tree:

```typescript
type MmlNode =
  | { kind: 'text'; text: string }
  | { kind: 'tag'; tag: string; attrs: Record<string, string>;
      children: MmlNode[] };
```

Replaces the prior flat-regex `MmlRenderer` (which broke on any `<`
inside a tag body — the inline TODO in the old file was the
regression). Handles arbitrary nesting; tolerates unclosed tags by
dropping the mismatched close; preserves all five MML entities
(`<` `>` `&` `"` `'`) and their decoding (mirrors server
`Mml.escape`).

### `MmlRenderer` — tree → React nodes

`packages/client/src/components/MmlRenderer.tsx` walks the tree
recursively. Each tag type has a styled-component or a click handler:

- **Clickable identity tags** (`<exit>`, `<detail>`, `<item>`,
  `<name>`, `<location>`, `<object>`, `<player>`, `<npc>`) wrap their
  children in `ClickableSpan`. The click target comes from
  `commandFor(node)`:
  - `<exit dir="X">` → `go X`
  - `<detail key="X">` → `look X`
  - identity tags with `stuff-id` → registry hit → `look <primaryKeyword>`;
    miss → `look <label>`
- **`<link href="…">`** dispatches by scheme: `mudcmd:` decodes the
  payload; `mudref:` resolves the stuff-id against the client stuff
  registry then `look <kw>` (or `look #id` on miss); `mudq:` is
  inert (rendered in `InertLinkSpan` — distinct from `ClickableSpan`,
  no underline, no cursor change, no hover state).
- **`<mention stuff-id="X">`** compares `X` to the viewer's `stuffId`
  and applies the stylesheet's `mention.match` (self) or
  `mention.other` treatment.
- **Inline emphasis** (`<strong>`, `<em>`, `<code>`, `<pre>`,
  `<blockquote>`, `<strike>`) maps to fixed styled-components.
- **`<chan>`** renders a chip with the resolved channel color (from
  the user overlay if set, else the theme default).
- **`<msg>`** is a region marker the per-message-type templates
  consume for layout; inline rendering passes through.
- **Unknown tags** render their children verbatim (forward-compat
  with future server-side tag additions).

### Per-message-type templates

`packages/client/src/lib/templates/` holds the registry and per-topic
templates. `pickTemplate(topic)` does longest-prefix dispatch:

| Topic family | Template | Layout |
|---|---|---|
| `speech.channel` | `chatTemplate` | gutter `<chan>` chip; content column with `<player>`/`<name>` + `:` + `<msg>` and hanging indent |
| `speech.vocal` | `sayTemplate` | inline: `<name> says, "<speech>"` |
| `speech.comms` | `tellTemplate` | inline directional treatment (sender → recipient); quieter than `say`. |
| `act.emote.*` | `emoteTemplate` | inline italic, action-shaped |
| anything else (including `system.*`) | `defaultTemplate` | body MML inline, theme-default treatment |

The `Terminal` component picks the template per frame and renders
through it. There is no `sys` template — `system.*` frames ride
`default`; whole-frame distinction (muted color, prefix marker) is a
stylesheet topic-cascade rule, not a layout template.

Adding a new template is purely client work — register it under
`packages/client/src/lib/templates/` and bind a topic prefix.

### Stylesheet engine

`packages/client/src/lib/style/Stylesheet.ts` resolves selector →
treatment lookups. Cascade order: **theme → user overlay → plain-mode
override** (which collapses every treatment to identity).

Five selector kinds:

| Selector | Source |
|---|---|
| `topic` (longest-prefix cascade) | `frame.topic` walked down its dots |
| `channel` | `<chan id>` attribute |
| `element` | tag name |
| `attribute → bucket` | `stuff-id` → `BucketResolver` → bucket name |
| `content-match` | viewer-relative (own-name / `<mention>` target) |

A `Treatment` is a small fixed-key object: `{ fg, bg, weight, italic,
prefix, chip, indent }`. Unknown keys are silently ignored; unknown
selector prefixes no-op silently. **No raw CSS reaches the engine.**

### Themes

Two themes ship in v1:

- **default** — `packages/client/src/lib/style/themes/default.ts`. The
  existing cockpit dark palette lifted into stylesheet form.
- **high-contrast** — `themes/highContrast.ts`. Accessibility-driven;
  strong fg/bg contrast, larger weight deltas, **every colored
  treatment paired with a non-color cue** (weight / prefix / chip /
  position). The high-contrast theme being legible without the color
  channel is the acceptance gate against color-alone semantics.

Theme entries' frame-level topic rules stay **empty for content
topics**. Per-word markers (`<strong>`, `<em>`, `<code>`) live at the
element level via the renderer's hardcoded styled-components — if the
frame-level topic rule claims the same dimension (bold, italic), the
per-word marker becomes invisible. Speech is already marked by the
quote characters in the body and the framing "You say,"; no extra
typographic claim is needed at the frame level.

### Font-by-register typography

The transcript renders **font-by-register**: world/social prose in a
proportional literary serif, the command/code register in monospace.
This is a **treatment on the existing topic cascade**, not a parallel
mechanism — and it is **client-only**: `StyleTreatment` is a shared
wire type with a fixed key set (no `font` key), so the font indirection
lives entirely on the client `Theme` via two sibling tables:

- **`Theme.registers: Record<string, FontRole>`** — the explicit
  topic-prefix → register map (`FontRole = 'narrative' | 'chrome' |
  'command'`). Resolved by the **same longest-prefix cascade** as
  `topicTreatment` (deepest matching prefix wins).
- **`Theme.fontRoles: Record<FontRole, string>`** — the role → CSS
  font-family stack. The swappable-faces layer.

`Stylesheet.fontFamilyForTopic(topic)` composes them: walk the dotted
topic, pick the deepest matching register, return `fontRoles[role]`.
Unmapped topics default to **`command` (mono)** — the conservative
MUD-throwback default, so an unclassified future topic stays mono.

The **three-voice model**: *serif = the world speaks · sans = the app
chrome · mono = you + the machine.* `world.*` prose maps to `narrative`
(serif); `system.*` and command echo (`shell.diagnostic.*`) map to
`command` (mono). The `chrome` (sans) role is the client-shell frame
voice — declared and self-hosted, but intentionally **not mapped to any
transcript topic**. Both shipped themes share the identical mapping
(typography register is orthogonal to the contrast axis); the tables
are factored into `themes/registers.ts` to avoid drift.

**Where the font is applied.** `Terminal.tsx` sets the per-frame family
on the `Body` ancestor (`stylesheet.fontFamilyForTopic(frame.topic)`),
so template spans inherit it with no per-template plumbing. `<pre>` /
`<code>` keep their element-level monospace (`MmlRenderer`) by CSS
specificity — an element's own `font-family` wins over an inherited
one. `line-height` stays on the `TerminalContainer` as the shared
cross-register **rhythm anchor** (never per-register on `Body`), so
switching register produces no baseline jolt. Plain mode does **not**
strip the register — font is structural legibility, not decoration, and
the failsafe message string is unchanged (reader sovereignty intact).

**The faces** are the Source superfamily — Source Serif 4 (narrative,
regular + italic), Source Sans 3 (chrome), Source Code Pro (command) —
self-hosted as subset OFL woff2 under `public/fonts/`, declared by the
`createGlobalStyle` `@font-face` block in `styles/GlobalFonts.ts`
(`font-display: swap`). `GlobalFonts` is mounted in `main.tsx` **outside
`React.StrictMode`** — a `createGlobalStyle` under StrictMode is injected
then removed by the simulated mount→unmount→remount and never re-added
(styled-components #3601), so its `@font-face` block silently never
lands. **No Google Fonts / third-party CDN request at runtime.**

**Single source, app-wide.** The three face stacks live once in
`styles/faces.ts` (`FACE_STACKS.serif/sans/mono`); both the transcript
register table (`Theme.fontRoles`) and the UI design tokens
(`components/ui/tokens.ts`) resolve through it, so a face swap is one
edit there. The three-voice model is applied **across the whole client**,
not just the transcript: `tokens.font.family` defaults to **sans** (the
chrome voice — nav, tabs, start/char-gen screens, menus, buttons, and the
inspection-pane chrome: breadcrumbs, header, Refresh), `GlobalFonts` sets
a `body` sans base for unstyled/portaled text, the command console
(`CommandBar`) and `<pre>`/`<code>` (`MmlRenderer`) opt into **mono**
(`tokens.font.mono` / `FACE_STACKS.mono`), and the transcript paints
**serif**/**mono** per register. The **inspection-pane body** (its
`Body` content — description, exits, contents) paints **serif** too, so
the pane reads like the transcript's look frame while its surrounding
chrome stays sans. Faces are a
swappable default theme, not load-bearing: re-skinning narrative to
Literata is editing the one `FACE_STACKS.serif` line plus dropping the
woff2 in — no controller, topic, template, token, or content change.

### Friend/foe stub

`packages/client/src/lib/style/BucketResolver.ts` exposes the
`BucketResolver` interface and a `NEUTRAL_BUCKET_RESOLVER` stub:

```typescript
interface BucketResolver {
  resolveBucket(stuffId: string): 'friend' | 'foe' | 'neutral';
}
```

v1 always returns `neutral`. The selector is wired and exercised but
inert. Swapping in a real implementation is one line when the
social-graph slate ships.

## The user overlay + the `style` verb

### Storage on `HasInteractiveMixin`

The reader-owned visual customization overlay is **one persisted JSON
blob per Avatar**, stored on `HasInteractiveMixin.clientState` under
key `'style.overlay'`. The schema entry's validator is permissive
(top-level "must be an object") — the resolver does per-value
defensive parsing so unknown selectors or treatment properties
silently no-op. This split keeps the validator simple and the runtime
forgiving for future GUI editors writing partial states mid-edit.

The overlay is **not** a setting. Per
[[settings-vs-propertied-vs-client-state]] the three persistence
categories are:

- **`EnvironmentMixin` settings** — player-tunable knobs via the
  `settings` shell command.
- **`PropertiedMixin`** — generic per-Stuff property bag.
- **`HasInteractiveMixin._clientState`** — client-UI-state held
  server-side; the overlay is the third category, sized larger than
  the per-tab / per-mute fragments that console-foundations landed.

### Selector vocabulary

| Key | Value |
|---|---|
| `theme` | `'default' \| 'high-contrast'` |
| `plain` | boolean (global plain-mode) |
| `plain.channel.<k>` | boolean (per-channel plain-mode) |
| `mention.self` | boolean (own-name highlight; default ON) |
| `channel.<k>.color` | string |
| `element.<tag>` | `StyleTreatment` |
| `topic.<prefix>` | `StyleTreatment` |
| `attribute.<attr>.<value>` | `StyleTreatment` |
| `mention.match` | `StyleTreatment` (self-mention) |
| `mention.other` | `StyleTreatment` (mention of someone else) |

`StyleTreatment` and `StyleOverlay` live in `@saxonberg/types`. The
shape is **bounded** — the engine recognizes a fixed set of keys; no
`position`, no `z-index`, no global escape hatches.

### The `style` verb

`StyleController` (`mud/obj/command/shell/StyleController.ts`) is the
player's surface for editing the overlay. Subcommands:

| Usage | Effect |
|---|---|
| `style show` | Print current overlay as readable JSON |
| `style theme <name>` | `default` \| `high-contrast` |
| `style channel <key> color <value>` | Per-channel chip color |
| `style channel <key> clear` | Drop all overlay rules for the channel |
| `style mention self on\|off` | Own-name highlight toggle |
| `style plain on\|off` | Global plain-mode toggle |
| `style plain channel <key> on\|off` | Per-channel plain-mode |
| `style reset` | Clear overlay to `{}` |

`style` is a **single-token verb** per [[no-two-word-verbs]];
subcommands ride argument shape. The controller travels with
`HasInteractiveMixin.commandContributions.self`, so wherever the mixin
composes the verb appears.

Every mutation:

1. `host.setClientState('style.overlay', next)` — schema-validated.
2. `host.save()` (fire-and-forget) — persists the new state.
3. `host.pushClientStateUpdate('style.overlay', next)` — push the
   authoritative value to connected Interactives.

### `client-state-update` push wire

Persistence + push are decoupled. The push half rides a new outbound
message:

```typescript
interface ClientStateUpdateMessage {
  type: 'client-state-update';
  payload: { key: string; value: unknown };
}
```

Parallel to the existing `client-state-write` inbound (client →
server) — but flipped direction (server → client). The server mutates
out-of-band; the client receives the authoritative value and
re-renders without waiting on the reconnect snapshot.

`HasInteractiveMixin.pushClientStateUpdate(key, value)` walks every
connected `Interactive` and calls a strategy-injected push function.
The injection pattern breaks an ESM load-time cycle:

- `Application.initialize` calls
  `setClientStateUpdatePush((interactive, key, value) =>
  this.sendClientStateUpdateToInteractive(interactive, key, value))`.
- `HasInteractiveMixin` stashes the function in a module-scoped slot
  and calls it from `pushClientStateUpdate`.

Direct import would create `Application → Avatar/Login →
HasInteractive → Application` — `Login` consumes `HasInteractiveMixin`
at module-eval time and breaks load. The setter pattern defers
resolution until after both modules finish loading.

On the client (`services/websocket.ts`), inbound messages whose
`type === 'client-state-update'` (no `frameId`) are dispatched
directly to `useStore.getState().setLocalClientState(key, value)`.
Bypasses the Sensor pipeline — this is wire-substrate plumbing, not
narrative.

## Acceptance shape (Wave 1)

The full criteria list lives in the retired requirements doc; the
sweep distilled them to:

- Every cockpit frame routes through one renderer.
- The body is a tagged-complete-string; flatten produces the failsafe.
- Four templates ship (chat / say / tell / emote); everything else
  goes through default.
- The stylesheet engine + two themes (default + high-contrast) work
  end-to-end via the `style` verb.
- Discord-dialect markdown subset parses server-side; `label`
  with the three custom schemes is the only link surface; unknown
  schemes strip.
- Mentions resolve server-side, silent on miss.
- Plain mode collapses every styled output to the failsafe.
- No `console.*` settings keys are introduced for visual
  customization — the `style` verb is the only player-facing UI.

## Cross-References

- [messaging.md](./messaging.md) — wire substrate (`MessageFrame`,
  Scene composer, `Mml.compose`, VocalMixin, Sensor pipeline).
- [response-envelope.md](./response-envelope.md) — `frameId` ordering
  primitive shared by both channels.
- [topics.md](./topics.md) — `TopicCatalogue` (authored source of
  truth for topic descriptors after `MessageApi.Topics` retirement).
- [connection.md](./connection.md) — `HasInteractiveMixin`,
  `_clientState` storage, welcome-payload snapshot,
  `client-state-update` outbound push.
- [inspection-pane.md](./inspection-pane.md) — consumes
  `MmlRenderer.commandFor` and the stuff registry the markdown
  parser's `mudref:` links resolve against.
- [shell-environment.md](./shell-environment.md) — `EnvironmentMixin`
  settings keyspace. **Explicitly NOT used** by the style overlay;
  cascade-shaped configuration is a different ergonomic.
- [command-spec.md](./command-spec.md) — pattern for adding a verb
  controller (the `style` verb follows it).
- docs/slates/tails/message-rendering-slate.md
  — seeding slate; Wave 2 (layout library) and Wave 3 (channel
  stylesheets) parked there.
- docs/slates/tails/social-graph-slate.md
  — real `BucketResolver` source (stubbed neutral in v1).
- docs/slates/tails/recognition-slate.md
  — `DescribeApi` v2 names; the renderer is forward-compat.
- docs/slates/tails/emotes-slate.md — future
  emote build; honors the SoulMixin + AetherMixin composition gate
  for remote emotes documented above.
- [docs/antipatterns.md](../antipatterns.md) — `no-new-apis-default`,
  `no-two-word-verbs`, `settings-vs-propertied-vs-client-state`
  (memory tags referenced above).
