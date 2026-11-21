# Tooltips

One tooltip for the whole client. A tooltip is a **small document that
previews something you could also click through to**: a thing's card, a
term's glossary page, or the command a click runs. It is never the only
place a fact lives. Shipped by the tooltips build (MR !633); the requirements
and plan retire at its sweep, and the open design space stays in
tooltip-slate.

## Tooltips and cards: two containers, one content

The owner: *"they're different things right? we might be tempted to unify
them but I don't think we should."* The rule is to **share what they say,
never how they live**.

| | a card | a tooltip |
|---|---|---|
| born from | a command — something ran | pointing |
| lives | in the feed; pinned means kept for good | while you point at it or hold it locked |
| changes | live while attended; every change names its cause | a snapshot taken at open; reopen it to refresh |
| is | a record of something that happened | a preview of something you could open |

- **No card controls on a tooltip.** Nothing pins it in place, docks it,
  minimizes it or keeps it live. A tooltip that stayed on screen and stayed
  current would be a card that no command opened, which is the
  "magic" problem the navigation-server review removed.
- **One content.** A thing's card renders the same fact rows and spoiler
  block as its tooltip, from the same builder (`thingDoc`) and the same
  blocks (`components/doc/blocks.tsx`). Neither container imports the
  other.
- **A tooltip opens no card.** Clicking the thing a tooltip is attached to
  opens its card, as it always did. A tooltip only shows what it says, and
  locks so you can read it and nest into terms. *(An earlier pin button,
  which ran the card's command, was cut in review as confusing and
  redundant with the click.)*

## The document

`TooltipDoc` (`@saxonberg/types`) is one wire and client type with these
blocks:

- **header** — icon, name, kind word;
- **facts** — each with its `source` (`seen` · `help` · `wiki` ·
  `client`) and when it was taken;
- **prose** — MML, which carries links;
- **image** — an illustration thumbnail, at most 96 px and never cropped;
- **spoilers** — the one spoiler block;
- **hint**, plus `command` (what a click on the anchor runs, boxed in the footer);
- **miss** — a line said instead of content, such as *You can't see that
  now.*, *Too dark to make out.* or *No help for that.* A miss is always
  said, never shown as an empty panel.

Inside a tooltip's prose, links follow a three-part grammar: a **term** is
dotted (hover it for a nested tooltip), a **thing** is underlined (hover for
its tooltip, click for its card), and a **command** is boxed. The scrollback
keeps its own link style.

## Three sources, one honesty rule

`services/tooltipSource.ts`:

- **A live thing** — a one-shot `mql-query` describe. The server enforces
  what it may say:
  - **Reach:** the one-shot keeps only subjects in the viewer's `reachable`
    set, which is `look`'s own scope, whatever the query text says. An id
    names a thing anywhere; describing it is an act, and acts observe.
  - **Light:** when the viewer's place is dark (read with `lightBandFor`,
    which learns and writes nothing), every field whose descriptor declares
    `sense: 'vision'` is dropped, and the envelope says `dark: true` once.
  - **Name:** the subject is named as the viewer perceives it
    (`describeFor`).

  The client caches each thing for 5 s and clears the cache when you move
  or the light changes.
- **A help term** — `GET /api/help/topic?word=` →
  `HelpApi.resolveTopic`, which applies the same order the `help <word>`
  verb uses. Cached for the session.
- **A wiki term** — the `wiki-excerpt` message → `WikiApi.excerpt` →
  `WikiRegistry.excerptFor`. It returns the page's opening section, rendered
  through **the wiki's own reveal gate** for the acting character: the
  ceiling deletes and the appetite collapses, exactly as `wiki <page>` does.
  There is no second reader of the wiki. A missing page and a page you may
  not read get the same answer (`doc: null`). Cached per page and appetite
  for 5 minutes.

What counts as a **term** is decided in one place (`termOf`): a link whose
command is `help …` or `wiki …`. So a new help topic or wiki page gets a
tooltip with no code.

## The spoiler block and the appetite

There is one spoiler rule, `useSpoilerOpen(level)`, shared by the tooltip
block and the terminal's `<spoiler>` span. A spoiler is **open when its
level is at or below the reader's appetite**; otherwise it is collapsed and
one click opens that occurrence. The appetite is the setting
`spoiler.appetite` (0–3, default 0; formerly `wiki.spoilerAppetite`). It
governs the wiki and every tooltip, and the client mirrors it
(shell-environment.md § the client mirror).

**A thing's path is a level-2 spoiler** (`TaggedPath`), shown under
*What is this, really?*. It is the **template** path, never the identity
path, which would put a player's id on every viewer's wire. The capability
ceiling never withholds a path; it is only ever collapsed.

## Gestures

- **Pointer:**
  - A tooltip opens after a 200 ms hover. It locks after another 400 ms, or
    at once on **Shift**, so the pointer can move into it.
  - A locked tooltip **stays where it opened**.
  - A term inside opens a child, and the locked stack shows a breadcrumb.
  - **Esc** closes one level. A press outside closes them all, and so does
    leaving every tooltip for 300 ms.
- **Keyboard:**
  - Focus opens a tooltip at once, already locked.
  - The transcript is **one tab stop**: ↑/↓ and Home/End move among its
    links, and Enter runs one.
- **Touch:**
  - A long press on a link, chip or control opens its tooltip as a
    **bottom sheet**. It does not open the scrollback row's line menu.
  - A tap still opens the command sheet.
  - Tapping a term inside the sheet pushes it onto the stack, with a back
    control.
  - A thing's sheet has an **actions** button that opens the affordance
    radial. A long press used to open the radial; on desktop, a mouse hold
    and right-click still do.
- **Disabled controls:** a disabled control gets no pointer events, so `Tip`
  attaches its tooltip (often the reason it is disabled) to a thin wrapper
  around it.
- **Status bar preview:** hovering still previews the command there.
  Tooltips never write `ghostPreview`.

## The pieces

| piece | where |
|---|---|
| the type | `TooltipDoc` & co., `packages/types` |
| blocks + builders | `components/doc/blocks.tsx` · `docs.ts` (`controlDoc`, `frameDoc`, `thingDoc`, `helpDoc`, `wayDoc`, `detailDoc`, `commandDoc`, `hoverDoc`, `termOf`) |
| the anchor | `components/tooltip/Tip.tsx` (`<Tip tip={…}>child</Tip>`), `useTip.ts` |
| the positioner | `components/tooltip/useFloat.ts` over `@floating-ui/react-dom`. The one positioner; the gutter menu and the tab-strip confirmation use it too |
| the stack | `store/tooltipSlice.ts`; `TooltipLayer.tsx`, mounted once around every screen in `App.tsx` |
| the sources | `services/tooltipSource.ts`; `websocketClient.query()` / `.wikiExcerpt()` |
| the gate | `src/__tests__/noNativeTitle.test.ts` — **no native `title` hover text anywhere in the client** (ceiling 0; an `<iframe title>` is an accessible name and is never counted) |

## Not here yet

- **Map glyph tooltips, breakdowns** (a number and why it is that number),
  **action buttons**, and **one block per mixin** all come with the
  navigation build, which is this component's next consumer.
- **Tooltips composed from the client's knowledge model** come with the
  same build's knowledge-model wave.
- **Live cards are not light-gated.** A card keeps its description in the
  dark, so a tooltip is currently more honest than a card. The attach
  point is the descriptor's `sense` key. Live subscriptions also keep
  describing a subject after you leave its reach, and cards rely on that;
  both are card-surface questions
  (spoiler-slate).
- **Portrait recognition** on a person's image waits on
  [media.md](./media.md)'s deferred recognition gate.

## History

- 2026-11-19 — shipped (MR !633, W0–W8): the component, the server's
  honest one-shot (reach and light), the gated wiki excerpt,
  `HelpApi.resolveTopic`, the `client` settings mirror and the
  `spoiler.appetite` rename. All 104 native hover strings were retired.
- 2026-11-19 — review:
  - **The pin was cut** (owner: *"that is so confusing … take it out"*):
    it borrowed the card's word and duplicated the click.
  - **Moving from one link to the next** now keeps the new tooltip; any
    anchor counts as the pointer staying.
- 2026-11-20 — the sweep, on a fresh world:
  - **Full suite:** 56/56 packages (20,386 tests).
  - **Lint family:** 118/118 gates.
  - **Wire drive** `tooltips.dirty.wire.test.ts`: 5/5.
  - **Browser walk** `tooltips-walk.spec.ts`: 5/5.
  - **Five defects the walk found** were fixed in the build:
    - a tooltip placed while it still showed "…";
    - a locked stack stuck open over a re-rendered anchor;
    - a pin ran an ambiguous keyword;
    - a red wiki link was read as a term;
    - help terms had no anchor in the world.
  - The boundary walk's *hut window* step fails because the glazier's hut
    stays pitch dark after `toDaylight`. That step, its content and its
    light were all untouched by this build.

