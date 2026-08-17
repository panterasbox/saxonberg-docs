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

⚠ **The cascade is unchanged by the custom-property colour layer.** The
ground is not a fourth cascade step; it is what the theme layer's values
*resolve to* — `Stylesheet.ts` is untouched, its tables still hold
strings, and the strings are now `var(--sx-…)` references. Selector
resolution, the fixed `Treatment` key set and "no raw CSS reaches the
engine" all hold. An overlay override still wins (it is returned earlier
in the same cascade), a player-typed `blue` or `#ff0000` is still a raw
string and still works, plain mode still returns `EMPTY`, and
`fontFamilyForTopic` still ignores `isPlain()`.

### Themes

Three themes ship, registered in `themes/index.ts`:

- **ink** — `packages/client/src/lib/style/themes/ink.ts`. The dark
  ground, and the default. A deepened navy drawn from Old Glory Blue,
  brass accent, hairline rules.
- **marble** — `themes/marble.ts`. The light ground.
- **high-contrast** — `themes/highContrast.ts`. Accessibility-driven,
  re-based on civic; strong fg/bg contrast, larger weight deltas,
  **every colored treatment paired with a non-color cue** (weight /
  prefix / chip / position). The high-contrast theme being legible
  without the color channel is the acceptance gate against color-alone
  semantics.

⚠ **`default` is retired and NOT aliased.** It stopped naming anything
the moment there were two grounds — the design doc calls the dark ground
Ink and so does the verb. `pickTheme('default')` falls through to Ink
like any other unknown string, and `cockpit style theme default` refuses
outright. (The S2 precedent: no playerbase to protect, the demo wipes
nightly, and an alias keeps a dead vocabulary alive for nobody.)

⭐ **Marble is not decoration — it is the verification.** A theme-aware
colour layer with exactly one theme is untested by construction: nothing
distinguishes a component that resolves at render time from one that
read a constant at import time. Marble is the second variant that proves
the abstraction, which is why it could not be added later "once we're
sure" — being sure is what it provides.

Theme entries' frame-level topic rules stay **empty for content
topics**. Per-word markers (`<strong>`, `<em>`, `<code>`) live at the
element level via the renderer's hardcoded styled-components — if the
frame-level topic rule claims the same dimension (bold, italic), the
per-word marker becomes invisible. Speech is already marked by the
quote characters in the body and the framing "You say,"; no extra
typographic claim is needed at the frame level.

### The custom-property colour layer

Before the civic build there were **two independently-authored colour
tables for one product**: the switchable `Theme.palette` (transcript
only) and the static `components/ui/tokens.ts` (read by 52 files).
Nothing connected them, so they could drift. They are now one.

**`styles/ground.ts`** declares the vocabulary and contains **no hex**:

| Export | What it is |
|---|---|
| `GROUND_ROLES` | 44 role names in five tiers. A name appears **once**; everything else derives from it |
| `Ground` | `Record<GroundRole, string>` — a theme's concrete values |
| `SX` | the `var()` **reference** table (`SX.surface === "var(--sx-surface)"`), built from `GROUND_ROLES` so a name cannot be typed twice |

The five tiers: **1** the 18 published ground roles, verbatim from
`DESIGN-SYSTEM.md`; **1b** the official colours + `ember` (4); **2**
derived chrome — `-lift`/`-press` steps, films, and the two stripe
percentages (11); **2b** the social tints, 1:1 with the server's
`NotifyRule.PaletteToken` vocabulary (8); **3** invariant platform marks
(3), declared once in `themes/invariant.ts` and spread into all three
grounds so no theme can repaint the flag or a third-party trademark.

Each theme supplies one `ground: Ground` record of hex. **Hex appears in
exactly one place per value** — that theme module — and every other
colour-bearing table holds an `SX.*` reference.

**Emission is imperative**, in `lib/style/useGround.ts`:

```ts
applyGround(ground: Ground): void;  // setProperty per role on :root
useGround(): void;                  // effect; re-applies on theme change
```

⚠ **Why imperative and not a second `createGlobalStyle`.** `GlobalFonts`
has to sit outside `React.StrictMode` because a `createGlobalStyle`
under StrictMode is injected, removed by the simulated
mount→unmount→remount, and never re-added (styled-components #3601).
Any new global-style sink inherits that trap. Emitting properties
imperatively sidesteps it entirely: nothing is injected, so nothing can
be removed and never re-added. Three consequences follow — `useGround`
can live *inside* StrictMode and read the store; `setProperty` is
idempotent so the double-invoked effect is harmless; and it is testable
in jsdom, which a `createGlobalStyle` sink is not. ⚠ **It deliberately
has no cleanup** — a cleanup on the simulated unmount is precisely how
#3601 bites.

`main.tsx` calls `applyGround(INK_THEME.ground)` before `createRoot`, so
first paint has a ground. `useGround()` is the **first line of `App()`**,
above the phase switch — the switch early-returns per connection phase,
so a hook inside one case would leave the other four unpainted.

`themes/index.ts` holds `THEMES` and `pickTheme`, and is **the
divergence guard**: `useStylesheet` (transcript) and `useGround`
(chrome) resolve the same overlay value through the same function. Two
local copies could drift and leave the transcript wearing one ground
inside chrome wearing another.

#### ⚠ The silent-failure mode, and the four guards

A missing or misspelled custom property does not throw, does not warn,
and does not render obviously wrong: **CSS drops the declaration and the
element inherits.** The page looks nearly right, which is the worst
outcome available. It is closed twice — `Record<GroundRole, string>`
makes a theme missing a role a *compile* error, and tests close the
rest. Under `lib/style/__tests__/`:

| Guard | What it closes |
|---|---|
| ⭐ `customProperties.test.ts` | definition (every theme covers the vocabulary) · **reference** (every `--sx-*` in source is a real role — the typo guard for a hand-written `var(--sx-surace)` that bypassed `SX`) · emission (the applier writes all 44, idempotently) · reverse (every role consumed, or reserved **with a reason**) |
| `noHexLiterals.test.ts` | no colour value outside `themes/`. Bans hex, `rgb()`/`hsl()` **without** a `var(` argument, and bare `white`/`black` in a colour position — a hex-only guard has three evasions and two were live |
| `contrast.test.ts` | every foreground pair against **`--surface`**, all three grounds, plus a ⭐ totality gate (a new role must be categorised) and the ⭐ red rule as an assertion |
| `oneColourSource.test.ts` | every token and every treatment fg/bg is `var(--sx-<role>)` — and, the other direction, a `ground` entry is never itself a `var()` |

⚠ Also **`e2e/tests/theme.spec.ts`**, which is not optional garnish:
**jsdom does not substitute `var()`**. `getComputedStyle(el).color`
returns the literal reference string, so no unit test can observe a
resolved colour. The e2e is the only place the cascade is seen end to
end, and it is what found that nothing set a root `color` — leaving
every inheriting element on the browser default black, 1.21:1 in
high-contrast. `contrast.test.ts` could not have caught that: it checks
values a theme *declares*, and this was a value no theme declared.

#### The `tokens.ts` alias table

`tokens.color`'s 21 keys keep their exact names and become `SX` reads,
so **zero call sites changed** across 1468 references:

| `tokens.color` | role | `tokens.color` | role |
|---|---|---|---|
| `surface` | `surface` | `primary` | `field` |
| `surfaceAlt` | `raised` | `primaryHover` | `field-lift` |
| `surfaceMuted` | `ground` | `primaryActive` | `field-press` |
| `surfaceSunken` | `sunken` | `border` | `line` |
| `fg` | `fg` | `borderMuted` | `line-soft` |
| `fgMuted` | `fg-mute` | `borderEmphasis` | `line-strong` |
| ⚠ `fgEmphasis` | `accent` | `sectionLabel` | `fg-mute` |
| ⚠ `accent` | `good` | `actionBg` | `raised` |
| `accentHover` | `good-lift` | `actionBgHover` | `line` |
| `onAccent` | `accent-ink` | `warning` | `warn` |
| | | ⭐ `danger` | `ember` |

⚠ **Two aliases cross over, deliberately.** The pre-civic palette was VS
Code dark, whose `fgEmphasis` was a gold (`#d7ba7d`) and whose `accent`
was a teal (`#4ec9b0`); civic's accent is brass and its `good` is
verdigris. Mapping by *appearance* rather than by name is what let every
call site stay put.

⭐ **`danger → ember`, not `danger → bad`, is how "red never touches
blue" becomes testable.** Ink's `--bad` is Old Glory Red at **2.66:1**
against Ink's `--surface` — unreadable as text on the navy field. That
is not a defect to fix by lightening the red; it *is* the constraint
stated as a measurement. `--ember` (`#e8705c`, 5.57:1) is the
alert-on-field colour. `bad` and `red` stay in the vocabulary for the
seal, the flag rule and the single committing action per screen —
surfaces that carry white separation. `contrast.test.ts` asserts that no
`tokens.*` alias resolves to `--sx-bad`, and that Ink's `bad` really
does measure below the floor (so the exemption's stated reason cannot
quietly become false).

Additive keys the hex sweep needed, with no call-site change: `info`,
`accentWash`, `onField`, `shadow`, `scrim`, `hatch`, `hatchStrong`,
`paper`, `paperInk`, `paperLine`, plus the `tokens.brand` group.

**Theme-invariant groups stay literal.** `space`, `radius`, the `font`
size scale, `rail` and `ratio` are the same in every ground —
`ratio.focal` in particular is a bare number in a `flex` shorthand, so
wrapping it would be actively wrong. Their *values* changed to the
published scale (radius 3px everywhere; spacing 4/6/9/12/16/22px).

**The one computed colour.** `GutterStripe.colorForTopic` hashes a topic
family to a hue. The hue must stay computed, but its saturation and
lightness were hand-tuned for the dark terminal — theme-blind. It now
composes rather than resolves: `hsl(${hue} var(--sx-stripe-s)
var(--sx-stripe-l))`. This is the single site the hex guard's
"colour function whose arguments include `var(`" carve-out exists for,
and it is a pattern permission, not a file allowlist.

### Font-by-register typography

The transcript renders **font-by-register**: world/social prose in a
proportional literary serif, the command/code register in monospace.
This is a **treatment on the existing topic cascade**, not a parallel
mechanism — and it is **client-only**: `StyleTreatment` is a shared
wire type with a fixed key set (no `font` key), so the font indirection
lives entirely on the client `Theme` via two sibling tables:

- **`Theme.registers: Record<string, FontRole>`** — the explicit
  topic-prefix → register map (`FontRole = 'narrative' | 'chrome' |
  'command' | 'display'`). Resolved by the **same longest-prefix
  cascade** as `topicTreatment` (deepest matching prefix wins).
- **`Theme.fontRoles: Record<FontRole, string>`** — the role → CSS
  font-family stack. The swappable-faces layer.

`Stylesheet.fontFamilyForTopic(topic)` composes them: walk the dotted
topic, pick the deepest matching register, return `fontRoles[role]`.
Unmapped topics default to **`command` (mono)** — the conservative
MUD-throwback default, so an unclassified future topic stays mono.

The **four-voice model**: *narrative = the world speaks · chrome = the
app · command = you + the machine · display = the engraved civic frame.*

⚠ **`BASE_REGISTERS` keys the seven topic ROOTS**, which is the payoff of
a tree that carries subject matter — the voice a frame speaks in follows
from what it is about, so one entry per root replaces the per-leaf table
this used to need:

| Root | Register |
|---|---|
| `speech` `act` `sense` `self` `publication` | `narrative` |
| `shell` `session` | `command` |

(An earlier revision of this section said `world.*` → narrative and
`system.*` → command. **Those roots died in the S2 corpus replacement**
and the prose outlived them by two builds; the table above is the
shipped one.)

`chrome` and `display` are the client-shell frame voices — declared and
self-hosted, but intentionally **mapped to no transcript topic**. A
topic can only acquire a voice by appearing in the register table, which
is what keeps an unclassified future topic falling back to `command`
rather than silently picking one up. All **three** themes share the
identical mapping (typography register is orthogonal to both the ground
and the contrast axis); the tables are factored into
`themes/registers.ts` to avoid drift, and `themes.test.ts` asserts the
identity across all three consumers.

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

**The faces** are the civic four from `DESIGN-SYSTEM.md`:

| Role | Face | Where |
|---|---|---|
| `display` | **Spectral** 500, uppercase, `letter-spacing:.19em` | Section labels, headings, wordmark, binomials |
| `chrome` | **Public Sans** | Everything chrome. The US federal design system's face — civic by provenance |
| `narrative` | **Newsreader** | World prose, plate captions |
| `command` | **IBM Plex Mono** | You and the machine |

⚠ **Only ever two voices on screen at once.**

Self-hosted as subset OFL woff2 under `public/fonts/`, declared by the
`createGlobalStyle` `@font-face` block in `styles/GlobalFonts.ts`
(`font-display: swap`). `GlobalFonts` is mounted in `main.tsx` **outside
`React.StrictMode`** — a `createGlobalStyle` under StrictMode is injected
then removed by the simulated mount→unmount→remount and never re-added
(styled-components #3601), so its `@font-face` block silently never
lands. **No Google Fonts / third-party CDN request at runtime.**
`DESIGN-SYSTEM.md` ships a Google Fonts `<link>`; adopting it would be a
regression for a product whose claim is that it is auditable.

⚠ **The metric-mismatch note.** The previous arrangement justified
`font-display: swap` with "the Source superfamily shares metrics, so the
swap is near-imperceptible." **Four unrelated families do not share
metrics**, so the swap now reflows. `swap` is kept anyway — a
non-blocking first paint is the stronger property, and `block` trades a
reflow for invisible text.

**Six files, four families.** Weight coverage is not uniform because
what upstream serves is not uniform, and the difference is load-bearing:

```
spectral-latin.woff2            Spectral 400        (static)
spectral-500-latin.woff2        Spectral 500        (static — a REAL face)
public-sans-latin.woff2         Public Sans 100–900 (VARIABLE, one file)
newsreader-latin.woff2          Newsreader 400
newsreader-italic-latin.woff2   Newsreader 400 italic
plex-mono-latin.woff2           IBM Plex Mono 400
```

- **Spectral 500 must be a real face.** Browsers synthesize bold only
  and round 500 down to 400, so a lone 400 would silently lose the
  engraved weight the design specifies. Measured in a browser: 500
  renders 4px wider than 400 at 40px, and both differ from the Georgia
  fallback — the check that it actually loaded rather than fell back.
- **Public Sans is one variable file**, declared `font-weight: 100 900`
  so `font-weight: 600` is a real interpolated instance. A second file
  for 600 would be the same bytes under another name.
- ⚠ **Newsreader must be requested without the `opsz` axis** — with it
  the face silently fails to load and falls back to Times.
  `Art Direction - Civic.dc.html`'s own `<link>` *does* carry
  `opsz,wght@0,6..72,400`: the reference art contains the trap
  `DESIGN-SYSTEM.md` warns about. Follow DESIGN-SYSTEM.
  `globalFonts.test.tsx` bans the axis in a tuple position.

#### The subsetting procedure

Recorded here so the next face swap is not a rediscovery — before this
build the only trace of how the Source subsets were made was a commit
message describing the *result*. **This is a one-off build-time fetch
and creates no runtime dependency**; the thing `globalFonts.test.tsx`
guards is the *runtime* request.

**Procedure A — no new toolchain.** All four families are OFL and
Google-hosted, and Google serves per-`unicode-range` pre-subset woff2
(the shipped files are exactly these). Ask for the CSS with a
woff2-capable UA, take the `latin` block, fetch its `fonts.gstatic.com`
URL:

```bash
curl -sH 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 \
  (KHTML, like Gecko) Chrome/120 Safari/537.36' \
  'https://fonts.googleapis.com/css2?family=Spectral:wght@400;500&display=swap'
# → find the @font-face whose unicode-range starts U+0000-00FF, curl its
#   src URL into packages/client/public/fonts/
```

⚠ Check whether the family is static or variable before deciding how
many files to ship: request two weights and compare the returned URLs.
Identical URLs mean one variable file (declare the range); different
URLs mean real static instances (ship both).

**Procedure B — fallback.** `pip install fonttools brotli`, then
`pyftsubset <family>.ttf --unicodes=U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+0304,U+0308,U+0329,U+2000-206F,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD --flavor=woff2 --layout-features='*' --output-file=<name>-latin.woff2`.

**OFL requires the copyright notice to travel with the font** —
`public/fonts/OFL.txt` carries all four families' notices plus the
shared licence text.

**Single source, app-wide.** The face stacks live once in
`styles/faces.ts`; both the transcript register table
(`Theme.fontRoles`) and the UI design tokens
(`components/ui/tokens.ts`) resolve through it, so a face swap is one
edit there. `FACE_STACKS.serif/sans/mono` remain as **aliases** (`sans →
chrome`, `serif → narrative`, `mono → command`) so no call site moved
when the model went from three voices to four.

The model applies **across the whole client**, not just the transcript:
`tokens.font.family` defaults to **chrome** (nav, tabs, start/char-gen
screens, menus, buttons, and the inspection-card chrome), `GlobalFonts`
sets a `body` chrome base for unstyled/portaled text, the command
console (`CommandBar`) and `<pre>`/`<code>` (`MmlRenderer`) opt into
**command**, and the transcript paints **narrative**/**command** per
register. The **inspection-card body** paints narrative too, so the card
reads like the transcript's look frame while its chrome stays chrome.

⚠ **`tokens.font.engraved` is the display FACE; `tokens.font.display` is
the 22px step in the size scale.** DESIGN-SYSTEM names the role
"Engraved capitals, display"; the two halves of that phrase take the two
keys rather than colliding on one.

Faces are a swappable default, not load-bearing: re-skinning narrative
to Literata is editing one `FACE_STACKS` line plus dropping the woff2
in — no controller, topic, template, token, or content change.

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
| `theme` | `'ink' \| 'marble' \| 'high-contrast'` |
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

### `cockpit style`

`StyleController` (`mud/obj/command/shell/StyleController.ts`) is the
player's surface for editing the overlay. Settings:

| Usage | Effect |
|---|---|
| `cockpit style show` | Print current overlay as readable JSON |
| `cockpit style theme <name>` | `ink` \| `marble` \| `high-contrast` |
| `cockpit style channel <key> color <value>` | Per-channel chip color |
| `cockpit style channel <key> clear` | Drop all overlay rules for the channel |
| `cockpit style mention self on\|off` | Own-name highlight toggle |
| `cockpit style plain on\|off` | Global plain-mode toggle |
| `cockpit style plain channel <key> on\|off` | Per-channel plain-mode |
| `cockpit style reset` | Clear overlay to `{}` |

⚠ **`style` was a standalone verb and is not any more.** It was absorbed
into the one `cockpit` verb along with `layout` and `mode`, because all
three wrote the same `cockpit.*` clientState keyspace — see
[cockpit.md](./cockpit.md). It is not kept as an alias: one name per
thing.

⚠ The settings above are **not** command subcommands. `cockpit style` is
already the subcommand and subcommands are one level deep framework-wide,
so they ride positional slots and `StyleController` dispatches on them —
which is why it carries an explicit `default:` that refuses an unknown
setting. Without it, `cockpit style bogus` would silently succeed.

The controller travels with
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
- The stylesheet engine + three themes (`ink`, `marble`,
  `high-contrast`) work
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
- [card-surface.md](./card-surface.md) — consumes
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

## History

- **The civic ground** (`c3ea1f07..ee4c4d02`, client-rebuild Wave 1
  Build A). Three shifts between plan and implementation that are worth
  keeping, because each was a fact the plan asserted and the build
  measured:
  - **Six woff2, not seven.** The plan projected one file per
    face/weight combination. Upstream is not uniform: Spectral is a
    *static* family (400 and 500 are two real files, and 500 must be
    real because browsers synthesize bold only and round 500 down),
    while Public Sans is a single *variable* file covering 100–900. A
    seventh file would have been the same bytes under another name.
    **Check static-vs-variable before deciding a file count** — request
    two weights and compare the returned URLs.
  - ⭐ **The root `color` defect, found only by driving.** Nothing in
    the client set a base `color`, so every element inheriting one
    resolved to the browser default black — the right-rail card tabs at
    **1.21:1** in high-contrast. Pre-existing; the token layer only made
    it visible. `contrast.test.ts` could not have caught it, because it
    checks values a theme *declares* and this was a value **no theme
    declared at all**. The general lesson: a guard over declared values
    is blind to an absent declaration, and only a real cascade over real
    DOM can see an inherited default.
  - **`font.engraved` vs `font.display`.** The plan named the display
    *face* and the 22px *size* step the same key. DESIGN-SYSTEM's own
    phrase is "Engraved capitals, display"; the two halves take the two
    keys.
