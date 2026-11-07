# Paper — the medium, the mill and the press

⭐ **The Document is the source of truth; paper is one way of showing it.**
A screen shows a document **live**; a printed sheet shows what the document
said **on the day it was printed**, and whoever holds the sheet reads it.
Everything below is that rung, the mill that makes the sheet, or the press
that puts a document onto it.

Built by the paper build (MR on `reqs/paper`, 2026-11). Packs:
`trade-papermaking` (`/trade/papermaking`), `trade-printing`
(`/trade/printing`); venues at Heart's Delight (the paper mill) and Terminus
(the Gazette's print shop).

## ⭐⭐ What paper adds to text, in a game made of text

Text is the platform's primitive: every message, card and release is text
the platform makes and moves for free (the owner, 2026-11-05). So paper is
never "the text"; it is worth something only for what interface text lacks:

1. **A text that is a thing** — it lies somewhere, belongs to somebody, is
   carried, lost, burnt, torn or stolen, and is read by whoever holds it.
2. **A text frozen in time** — paper says what the document said that day;
   it goes stale, contradicts the present, and is evidence.
3. **A text with marks of its making** — a hand, a device, a misprint.
4. **A text that reaches where the network does not** — carried, at a
   walker's speed, into places and epochs with no screen.

Every piece below serves one of these. A design that makes paper a second
way to be READ — competing with the platform's own publishing — is wrong
by this test. Printing, the cost of copying such a thing, has its own
slate (printing-slate).

## Reading: two faces, one verb

- **`read`** takes a written thing (`MarkedMixin`) or a screen
  (`DisplayMixin`). A screen is read off what it shows (no light gate — it
  is its own light); a written thing is perceived, decoded, read.
- **Derived marks.** `Marked.refreshMarks(viewer)` (`@hook`, harbors' too)
  runs before a read: a printed sheet reads its edition, a book its open
  page, a ledger its house's papers. `getMarkText()` stays sync.
- **What reading teaches.** `Marked.teach(reader)` (`@hook`) runs after the
  text lands: a chart writes its places into the map; a **printed** recipe
  writes *known-of* (you know the dish exists, and can recognize it). ⚠ A
  handwritten page teaches nothing by being read, and nothing here decides
  how a recipe is held or consulted — that is the
  recipes slate's (your own recipe in
  your book, consulted when you `make`).
- **Ash.** A burnt marked thing (its material swapped to its char) reads
  *"nothing but ash"* — the ink rule's shape.

## The frozen rung — editions

- Document kind **`edition`** at `<source>/editions/<hash of the pages>`,
  written only by **`DocumentApi.freezeEdition`**: path pinned, kind pinned,
  owner derived from the source (or the records under a register prefix),
  idempotent by the pages, throws on a source that names nothing.
- **`DocumentApi.pagesOf(path)`** renders any stored document as pages,
  exhaustive over the closed kind table (a new kind is a compile error);
  `null` for a kind that does not read as a page.
- An edition freezes the text **as set** (a forme's misprints included);
  an edit or retraction of the source never reaches it.
- **Access by possession**: a sheet reads its edition with no title
  check. Printing a private record is the printer's risk.

## The live rung — screens

- `DisplaySource` arm **`document`**; `readScreen` renders it through
  `pagesOf` at read time — an edit shows, a retraction reads *"withdrawn by
  its publisher"*.
- **`show <what> on <screen>`**: `DocumentApi.callUp(reader, word)` — the
  newest standing release whose headline or publisher answers to the word,
  else a paper of the reader's own. The screen's pairing decides who drives.
- ⚠ Coverage (a handset off grid) is the off-grid build's; every screen
  fetches today.

## The sheet, the book, the seal

- **`Sheet`** (`platform/thing/Sheet.ts`): folds sheet → folio → quarto →
  octavo and no further; writing wants **ink in hand** and **size** (an
  unsized sheet bleeds; a set coat of a `size`-tagged material, or
  `sized: true` on the row); soaked through it gives way (below); ⭐ what
  is written on it keeps its HAND (`passages` — a sheet is a leaf);
  torn, the halves split between passages (each keeping its hands) or down
  the middle of printed words, bound to nothing, and the second half is the
  sheet's owner's (made from it); `markText` is stack identity, and the
  same words in two hands never stack.
  A mill's sheet carries its making: `color` (the vat's water), `evenness`
  (the vatman's band), `formed`, `pressed` — formed and dried unpressed it
  is `isCockled()`.
- **`Book`** (`platform/thing/Book.ts`) — ⭐ **a book is its leaves, and
  a leaf is a sheet still bound** (the owner, 2026-11-04/05). The assembly record carries
  the physical set (the leaves' paper, the boards, the spine); the book
  keeps one record per leaf (`leaves`): **passages, each with the hand that
  wrote it**, or the edition page it was printed with. There is no document
  behind a notebook — handwriting lives on the paper. Whoever holds it
  writes (ink in hand, sized leaves, room on the page); `read` names each
  hand as the reader knows its writer (*your hand*, *Hester's hand*, *a
  hand you do not know* — `Marked.readFor`, the device mark's rule).
  `read` / `write` / `tear <book> page <n>`. A row authors a **printed
  book** by naming its edition (`edition:`, seeded at mint) — the Gazette's
  Terminus Directory. **Tearing a page out** takes the leaf whole into a
  loose `Sheet` in the tearer's hand — its passages and hands, or its
  edition page — named *a torn-out page*, saying *a page torn from …*, and
  **still the book's owner's paper** (made from the book; a book nobody
  holds title to, like an authored prop, makes no claim, so the page is
  the tearer's — the chattel rule). A stub is left, and a look at the book
  names the pages gone. `read`'s target is a greedy name bounded by
  `page <n>` (`read city directory page 3`, `read torn-out page`).
  ⚠ Binding sheets into a book (`fit notebook`) is deferred; so are
  leafing for a word (`read … for`), a compiled directory and who is
  listed in it — the slate's § 12.
- **`RegisterBook`**: a fixed book that holds nothing — `registerPrefix`
  names the branch it lists on each read (the Gazette's feed; a
  government's register of ships). ⭐ What a line says is the record's
  kind: a ship's entry reads as her name, flag, company and shareholders
  (harbors' line), any other paper as its kind, leaf and title. ⭐⭐ The
  rule it embodies (the owner, 2026-11-05, reconciling harbors): **an institution's records are documents; the
  book on the desk is their page** — the harbor's berth book follows it
  too. A person's own handwriting is the other half: it lives on
  the leaf.
- **`Envelope`**: shut, it vetoes in and out; `seal` (gum or wax — wax wants
  wax in hand). On `Sealable`: `seal`, `sealBroken` — opening breaks it,
  closing never mends it, a look says so.
- **`TearableMixin`** + `tear`: sheet, book, garment (rags, mass for mass,
  not while worn), envelope, `Packet` (spills).
- **`Ticket extends Sheet`.**

## Paper fails by its material

⭐ There is no paper-goods mixin: **"made of paper" is a material.** Paper and
board keep about a tenth of their strength wet (`Material.wetStrength`), and
`Integrity` — the shipped home of how a good fails (the impact rule) — gives
way any good of a weak-wet material soaked through (`afterWetted`, the
`soaked` band), leaving its `soddenTemplate` (a lump of pulp). A notebook, an
envelope, a bag, a sheet and a spellbook alike, with nothing on their classes.
Burning is `Combustible` with the material's ignition point; tearing stays a
protocol each thing implements (`Tearable`), because what tearing makes is
the thing's own business.

## Paper's representations

Paper's version of the forestry four-representations pattern (a tree is a
place, a slot, a record or a prop by scale and use):

| rung | example | the act between |
|---|---|---|
| bulk | pulp in the vat | `dip` → a sheet |
| stack | a ream of blank sheets | `split` / merge (written words keep a sheet out) |
| thing | a letter, a notebook, a copy | `fit … with <pot>` → a coat |
| a coat on a face | a pasted label, a poster | wallpaper waits on walls having faces (R4) |
| a view at a fixture | the ledger | — |

There is no "big paper" as a place; paper's large scale is the face coat.

## Bearer paper

`ReceiptTerms.bearer` + `Store.receiptPaper`: a bearer deposit freezes the
receipt as an edition and hands the depositor a sheet bound to it; a bearer
withdraw presents the sheet (in hand or one wrapper down, never through a
seal). The depositor without it is refused; a settled one is answered so
and stays in hand; a hand copy (bound to nothing) presents nothing. The
depot's warehouse is bearer; the ice houses stay registered.

## The epoch covenant

`SpatialZone.admitsEpochThrough` (inherited by `lookupField`) and
`Tool.admittedAt(place)` — the first reader of the `epoch` stamp. A medieval
covenant keeps the press (industrial) and the printer (modern) idle and
admits the pen.

## The mill (`trade-papermaking`)

`Stamper` (switch on; beats clean rags to pulp at the race's power for the
hours it ran), `PulpVat` (the bath affords `dip`: a wet sheet on the
mould's wire, colored by the vat's reach — a persistent load makes brown),
`PaperMould` + `couch` (onto the post: the press bed), the press squeezes
the post, the loft dries it, the size tub sizes it. The mill buys clean
rags; a paint rag is another row. `Packet` (bag, carton); labels pasted
with `fit <label> to <good> with <pot>` (`Faced.pasteOn`), let go in the
wet unless the glue resists damp (`Faced.releaseDampGlue`).

## The press (`trade-printing`) — ⚠ a PLACEHOLDER

⚠ **Printing shipped as what paper implies, not as a designed trade**
(the owner, 2026-11-05). Every act below is instant, which hides the
fixed-versus-marginal curve that IS printing; the printing build replaces
this section (printing-slate — its
lens-decided forks: labor takes time, misprints stay because the document
is never altered, images are half of printing, and a printed page should
LOOK like one).


`TypeCase` (the font, capital), `Forme` (`compose` debits the case and
seeds the compositor's errors; `proof` corrects a seeded share; nobody is
told what was missed; `distribute`), `Press` (pulls the forme as set, ink
by the page), `Printer` (a document as it stands), `Pen` (`copy` by hand,
with the copyist's slips, bound to nothing). Misprints are **seeded, never
drawn** (`lib/Misprint.ts`, docs/uncertainty.md).

## Deferred

Retting, the stamper's fouling as a discharge, brown paper in the drive (no
persistent discharge exists), pasteboard/carton/book **bills** and `fit
notebook`, wallpaper (R4), filter paper (R3), parchment (R6), iron-gall from
galls, type metal and casting sorts, the woodcut handbill, the almanac's
year, the shop counter selling copies, the handset. See the paper plan's
wave notes and the paper slate.
