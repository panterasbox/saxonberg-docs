# Harbor — the port, its berth book, and what a ship owes

A harbor is where ships are **kept**: a set of quays somebody keeps a book
for, a harbormaster who gives berths by posted rules, dues that run while
a ship lies alongside, and an office that answers when a ship hails it.
Everything here belongs to the **`/system/transport`** pack. The kernel
reaches the harbor through two protocols on the places involved
(`Berthing` on a quay, `Hailable` on an office) and one on a book
(`Payable`). The ship never imports the pack.

This doc is the harbor's side. The ship's side (what she is, who owns her,
how she leaks and sinks) is [ship.md](./ship.md). How the ropes go ashore
and the way aboard appears is the expanse's ([expanse.md §
Exits](./expanse.md)). The bar's tide window is in [expanse.md §
Harbors](./expanse.md) and [river.md](./river.md).

⭐ **The harbor detains the SHIP, never the people.** Nothing here is asked
when somebody goes ashore or comes aboard. A ship that owes dues cannot
sail. Her crew can walk off her.

## The pieces

| piece | class (pack) | what it is |
|---|---|---|
| the harbor | `Harbor` (`/system/transport/idea/Harbor`) = `Registrar(Singleton(Idea))`, one row per port | the **posted rules** (quays, rates, seat, office, roads, exempt) and how they are applied. It holds no state; it **files** the berth book's record. |
| the berth book | `HarborBook` (`/system/transport/thing/HarborBook`) = `Marked(Payable(Good))`, a fixture on the office desk | where **the record** is kept and read: lines (berths given, ships alongside, debts, dues stamped paid) and the last hail of each ship. The record itself is a `berth-book` document the harbor files. |
| a kept quay | `HarborQuay` (`/system/transport/location/HarborQuay`) = `Berthing(Quay)` | a quay where a ship comes alongside only at the berth she was given; it asks its `harbor:` |
| the office | `HarborOffice` (`/system/transport/location/HarborOffice`) = `Payable(Hailable(FrameSited(Locale)))` | the room a hail reaches and where dues are paid; sited on the river with `frameSite.landing: false` |
| the berth roll | a `berth-roll` **document** at `/platform/berths<landing path>` (kernel) | a landing's list of who lies at it, each ship writing her own line. It is not the book (see below). |
| the register | `RegisterBook` (kernel) on the office desk | `read register`. See [ship.md § The register](./ship.md). |
| the charter book | `CharterBook` (kernel) at the line's desk and in a ship's hold | `charter` (below) |

A river's public landing (`Quay`, `Shore`) composes none of this and
stays first-come: any ship may lie there and go. A ship left asleep at an
unkept landing is charged by nobody (see Deferred).

## ⭐⭐ The rules on the row, the record filed, the book its page

`Harbor` keeps only what is **posted**, and its authored fields are all of
it:

- `quays: [{place, lengthM, berthDepthM?, maxDraftM?}]`, in the order the
  office gives them. `berthDepthM` is the water at the berth (a mud berth;
  the harbor's own fact until a dredger exists). `maxDraftM` keeps a
  shallow river berth for river craft.
- `office`, `seat: {business, positionKey}`, `roads` (a sea node),
  `rates: {duesPerMeterPerDay}`, `exempt: [ship handles]`.
- `book` names its berth book **forward**. It is followed through
  `StuffApi.singleton`, never searched for. A harbor with no book, or one
  whose book is kept for another harbor (`HarborBook.harbor`), throws.

`HarborBook` keeps the record and knows nothing of the criteria. It is
the fixture players act on (`look book`, `read book`, `pay book`,
`berth … at`) and the thing an author places.

⭐⭐ **The record is filed, not held** (the paper build, 2026-11-05 — an
institution's records are documents and the book on the desk is their
page; paper's ledger and the ship register work the same way). The lines
and hails are ONE `berth-book` document at `<registerPrefix>/record` in the
harbor's own branch (`registerOwner: /world/terminus/harbor`,
`registerPrefix: …/berths`, `registerKind: berth-book` on the harbor row).
The harbor files it (`Harbor.fileRecord` → `DocumentApi.saveToRegister`,
gated to the harbor's own book — *you file; you do not hold the pen*).
The book loads it on use (`loadRecord`: the harbor's `liveBook`, a
`read`, the book's own creation) and writes every change through, in
order. ⭐ **Burn or lose the book and no debt is forgiven:** the next book
on the desk reads the same record. The harbor still holds no state of its
own (review 2 stands); the book is no longer a persistence host. Its
working fields are `lines: BerthRecord[]` and `hails: HailRecord[]`:

- **`BerthRecord`** `{ship, name, lengthM, quay, assignedAtS, sinceS,
  paidThroughS, leftAtS?}`. `sinceS` is `null` while she is still coming
  in. `leftAtS` marks a line kept as a **debt**: she cast off owing, the
  line holds no room, and its days stop there.
- **`HailRecord`** `{ship, name, atS, quay, criterion, words}`. This is
  what she was last told. It lets the harbormaster name a ship lying in
  the roads, and it is shown on the book's page.
- An assignment she never takes up, and a hail, both lapse after 12 hours
  (grain, `ASSIGNMENT_HOLDS_S`).

The book's `look` page is derived from its own lines (a
`markupAugmenter`). A second port is a second harbor row and a second
book, with no code (see *A second harbor* below).

⭐ *Why the record moved* (review 2, `0f11a60ff`): the record had been a
persisted `book` field on the placeless `Harbor` Idea. Lens 2 put it on
the thing players touch and authors place. The Idea now holds the rules
only.

## Berthing is `course <quay>`

There is no mooring verb. A ship is brought to a kept quay the way any
ship is brought anywhere:

1. **Hail the office** (`hail office`). The harbor applies its criteria
   and either writes her into the book for a quay or sends her to the
   roads, citing the criterion (below).
2. **`course <quay>`** (on the river, the quay's own word, e.g. `course
   town quay`. At sea, `course <node>` for a harbor behind a node; from
   the river-mouth node, `course <quay>` goes in over the bar and up to
   it). Before the course is laid, `Ship.courseRefusal` asks the quay
   (`Berthing.admit`), so a refusal is instant and nothing moves: *"The
   harbor office has not given her a berth. Hail the office first —
   `hail office`."* A chart course for a node does not ask `admit` (the
   island is beyond hail from Terminus). The arriving stop asks it
   (`Ship.arriveAlongside`), and a ship not given the berth **lies off**
   and is told to hail.
3. **She makes fast** (`Ship.comeAlongside`). Each of her mooring lines
   (a rope on her deck kept at her bitts) is tied round a `Bollard` on
   the quay with a clove hitch, through the rope's own `tieTo`. A way
   aboard named for her (`BerthExit`, `go hesper`) goes on the quay. The
   quay is told `madeFast` and her line in the book starts its days. A
   charter running on her ends here (below). The full rope mechanics are
   in [expanse.md § Exits](./expanse.md) and [ship.md §
   Alongside](./ship.md).
4. **She casts off** when any course is laid from a moored ship (her
   lines come in, the way comes off), or when a person `untie`s her last
   line (cordage's act). The quay is told `release`. If she leaves owing,
   her line stays in the book as a debt.

`Berthing` (`lib/expanse/Berthing.ts`) is the kernel's whole view of a
kept quay: `admit`, `madeFast`, `clearance`, `release`, `berthDepthM`.
`HarborQuay` forwards each to its harbor.

⭐ **The berth is a way named for the ship**, wired when she makes fast and
removed when she casts off. Two ships at one quay are two ways (`go
hesper`, `go kittiwake`). The way is never mapped (`isMapped() → false`):
it leaves with her. The quay's `look` reads a derived line, *Alongside:
the Hesper, the Kittiwake.*, off those ways (`FrameSitedMixin`'s
augmenter). Nothing is a thing in the
room except the rope ends round the bollards: *"A mooring line comes
ashore from the Hesper, made fast round the bollard with a clove hitch."*

## ⭐ Two lists, two questions: the berth book and the berth roll

| | the berth book (the `berth-book` document) | the berth roll (`berth-roll` document) |
|---|---|---|
| answers | *who was given which quay, since when, and what is owed* | *which ships lie made fast at this landing* |
| exists at | kept quays only (a harbor's book) | every landing a ship makes fast at, kept or not |
| written by | the harbor, applying its rules | **each ship, about herself only** (`DocumentApi.saveBerth(ship, landingPath)`, gated `FromMixin(Voyaging, caller === args[0])`; the store reads `ship.getBerth()` off her, so she cannot write a false line) |
| read by | the harbormaster, `pay`, the book's page | the landing on first use: `FrameSited.lyingHere()` → `DocumentApi.berthRollOf`; every listed ship not loaded is stood up (`ExpanseApi.standUpCraft`) and re-wires her way; `Voyaging.answerBerthRoll` lets a ship that is no longer there strike herself |
| why | dues and berths are the port's business | after a restart nothing else attaches a sleeping ship of a kind to a landing, and a landing must not become a persistence host (its rope ends are re-tied from the ship's record) |

The roll lives at `/platform/berths<landing path>`, `owner` = the
landing's path, `data: {lying: [handle…]}`. Writes to one roll are chained
per path. See [document-store.md § `berth-roll`](./document-store.md).

## The harbormaster's seat, and the criteria in order

The harbormaster is a **seat**, not a class: a position on a business
(Terminus: `harbormaster` on the harbor works,
`/world/terminus/wharfside/idea/harbor-works`, headcount 1, `purchases:
true`). Its roster has an NPC (Ruth Harrow, stationed at the office 6–20
every day). ⭐ **The roster is the cover**: a player appointed to the seat
holds it beside the NPC, and **the office is open when any holder is on
shift** (`Harbor.isOpen` / `onShift`). There is no separate cover
mechanism.

A hail is answered by the posted criteria, **in order**, and the first
that fails sends her to the roads, citing itself (`HarborCriterion`):

1. **`closed`**: nobody holds the seat on shift. *"The harbor office is
   closed. Lie in the roads until the office opens in about N hours."*
2. **`bar`**: the water over the bar at the tide against her draft (the
   same number `course` reads; see [expanse.md](./expanse.md)).
3. **`no-room`**: no quay has room for her length. Ships fit end to end:
   a quay's length less every ship the book gives it, and less every ship
   made fast there by her own way that the book does not name (an exempt
   city craft). A quay's `maxDraftM` is also checked.
4. **`dues`**: she owes dues.

Otherwise the first quay with room is written into the book and she is
told the course to it. A ship already made fast is told she lies there.
Nothing is rolled. Nothing but the clock and the book changes the answer.

**A player keeper** (`berth <ship> at <quay>`, `berth <ship> roads`, at the
book, in the transport pack's `harbor` command category) has exactly the
NPC's authority and no more. `Harbor.keptBy` (holds the seat, on shift)
is checked before anything is looked up: *"You do not keep this harbor."*
`assignBy` refuses any quay the criteria would refuse (bar, draft, room,
dues), and a ship made fast is never moved. `sendToRoads` takes back only
a berth given and not yet taken up (criterion `keeper`). Those aboard her
are signaled. A player holding the seat on shift is also told of each
hail as it is answered (`Harbor.tellKeepers`). ⭐ A seat-holder has no
write path to the rules: they are the row's data, posted by whoever holds
title over the row through the CMS.

## The bar — the tide's window

A ship from the sea crosses the river's bar to reach the port, and the
bar is read, never rolled: the water over it now
(`LineFrame.barWater(nowS)`, the channel's depth at the mouth with the
tide's level in it) against her draft as she floats plus the margin a
master keeps under her keel (`UNDER_KEEL_M`, 0.5 m, a constant in
`Ship.ts`). `Ship.barClearance` is that difference. `Ship.barRefusal`
says it in a pilot's words with the hours to the next high water: *"There
is 2.4 m over the bar and she wants 4.1. Not until the flood — high water
in about 3 hours."* `course` asks it before she goes in, and the
harbor's `bar` criterion cites the same sentence. Detail:
[expanse.md § Harbors](./expanse.md), [river.md](./river.md).

## Hail

`hail <word>` tries **hailables** first, then the sails in sight.
`Hailable` (`lib/expanse/Hailable.ts`: `hailName`, `answersTo`,
`answerHail`) is composed by `HarborOffice`. `Frame.hailables(pos)` finds
them, loading only a place whose row's class composes `Hailable`:

- **on a river**, the places its line names within `expanse.signalRangeNm`;
- **at sea**, through each node in range: a river mouth hands the hail up
  the river, a node landing that answers itself, and the station a node's
  row names (`ExpanseNode.hail`, the way a chart lists a port's signal
  station). Waponi Woo's node names the estate store.

The office's answer is the harbor's (`Harbor.answerHail` →
`criteriaAnswer`): *"The harbor office answers: …"*. The hail is written in
the book. A harbor on a lake with neither a river line nor a node is not
reachable by `hail` (Deferred).

## Dues, clearance and `pay`

**Dues** run per world's day alongside, by her length, at the posted rate:
`ceil(days begun since paidThroughS) × lengthM × duesPerMeterPerDay`,
totaled and ceiled to whole minor units (`Harbor.duesOwedIn`). The stamp
is on the **book's line**, never on her, so a ship nobody has loaded since
she made fast still owes every day begun. Exempt ships owe nothing.

**Clearance**: before a moored ship's course is laid, the quay she lies
at is asked `clearance` (`Ship.courseRefusal`). The harbor refuses while
she owes dues **or** carries unpaid bills (`ship.owed`, e.g. a charter's
days): *"She owes 7 in harbor dues and 6 on her bills, and is not cleared
to sail until it is paid at the harbor office — `pay office`. Anyone aboard
may still go ashore."* People are never gated.

**`pay office`** (or `pay book`): both `HarborOffice` and `HarborBook`
compose `Payable` (`lib/banking/Payable.ts`) and forward to
`Harbor.billsFor`/`onPaid`. `pay <it>` with no amount pays every bill the
thing holds for the payer, in order, each through
`EmploymentApi.settleSale` to the venue it names (credential, then cash),
and stops at the first that cannot be paid. A stated amount at a `Payable`
is refused. A bill is **derived at the moment of paying**: for every ship
in the book that the payer **answers for** (`Ship.answersFor`: her master,
her charterer, her managing owner, or whoever owes one of her bills), her
dues go to the harbor works first, then each bill she carries. Paid dues
are stamped forward on the book (`stampPaid`). Paid bills are settled off
the ship (`Ship.settleOwed`). Bills settle with `taxable 0` (a public fee).

## The charter

A whole ship is hired for a voyage with `charter [<ship>]` (kernel,
`platform/cmd/work/charter.yaml`, `CharterController`), afforded by a
**charter book** (`platform/thing/CharterBook`, a fixture):

- **aboard** (the book in the Hesper's hold, `ships:` empty) it charters
  the ship it is in;
- **at a desk** (the Greywater Line's desk on the Town Quay) it names the
  ships it lets (`ships:`), and `charter <ship>` picks one.

The book posts `ratePerDay` (with a per-ship override in `rates`) and
`payTo` (the venue the days are paid into, the line's desk). A rate with
no `payTo` throws. The charter is the **ship's** record
(`Ship.charterTo` → `charter: {charterer, sinceS, ratePerDay, payTo}`).
It is refused if she is chartered already or her own master is aboard on
shift. While it runs, the charterer takes her helm
(`Ship.mayBeSteeredBy`).

⭐ **It ends as she makes fast at a kept quay** (a `Berthing` landing; a
plain landing is a stop, not a berth, so a charterer who leaves her at the
estuary mouth still has her). The world's days it lasted (a day begun is
a day owed) × the rate become a bill on her (`ship.owed`, `by` = the
charterer), and the charterer is told: *"The charter of the Hesper ends as
she makes fast: 3 days at 2 — 6 owed to the line, paid at the harbor
office."* Her clearance waits on it.

## Rates (grain)

Every number is a row field an author may change (lens 2, grain not law).
They were sized against the harbor works' wage (4 minor a game-hour, about
56 a 14-hour shift) so that *a smack's weekend away costs about one
shift's wage*, and so that a charter does not eat the sugar loop's bounty.

| rate | value | where |
|---|---|---|
| Terminus dues | 0.25 / m / day (a 9 m smack ≈ 2.25/day; the 30 m Hesper ≈ 7.5) | `harbor.yaml` `rates` |
| Waponi Woo jetty dues | 0.05 / m / day (the Hesper ≈ 1.5/day) | the island's `harbor.yaml` |
| charter: Hesper, Kittiwake | 2 / day | the charter books |
| charter: Corrigan | 3 / day | line desk's book `rates` |
| a smack off the yard | 5,000 | the hull book ([ship.md](./ship.md)) |
| rowboat / canoe, sold at the put-in | 160 / 110 | the put-in counter `prices` |
| hire: canoe / rowboat | 3 / 5 a day | the put-in hire book `rates` |
| chandler: caulker's kit, mooring line, oakum, pitch cask | 12, 20, 2, 8 | Hobb's counter `prices` |

## A second harbor is rows

Waponi Woo's jetty is the second instance, with **zero code**: its
landing row is a `HarborQuay` with a `frameSite` on the Greywater node
and `harbor: /world/terminus/waponi-woo/idea/harbor`. The estate store is
a `HarborOffice`. A berth book lies on the store counter. A `Harbor` row
lists one 40 m quay, the store as the office, a `harbormaster` seat on the
estate's business held by its keeper, and the island's own node as its
roads. The Greywater node row names the store as its `hail:` station, so
a ship lying off the island reaches it.

## The Port of Terminus

The district is `/world/terminus/harbor/` (a `CartesianZone` in the
terminus pack), on the Kestrel's left bank inside the bar, between the
estuary mouth and the reach. It is joined to the estuary by one edge (the
mouth's `south` → the Town Quay).

- **Town Quay** (70 m, berth depth 1.5 m) and **Fish Quay** (25 m, 1.2 m),
  with bollards; the **Harbor Stairs** (a `Shore` with a way into the
  water and a ring for a boat's bow line).
- **The harbor office** (`office.yaml`): the berth book, the register book,
  the harbormaster.
- **The line's desk** (the Greywater Line): the charter book naming the
  Hesper and the two regulars, the **Kittiwake** (a ketch, 20 m) and the
  **Corrigan** (a coaster, 35 m, draft 3.6 m: refused at low water,
  admitted at high), each with a company and an NPC master rostered on
  deck.
- **The yard** (`yard/`): its quay (15 m, a kept quay; a smack owes dues
  from the day she is sold), the shed and its hull book, a shipwright.
  The sale is in [ship.md § Buying a ship](./ship.md).
- **Hobb's**, the ship chandler, off the Town Quay's southwest.
- The **Wharfside bank** (60 m, 0.8 m, `maxDraftM` 0.6) is a quay of the
  same book. The **Ostrander**, the city's river service, is `exempt`.

The berth depths are authored at or under a smack's hull (1.8 m), because
the derived channel depth at the quays (2.0–5.9 m across the tides) would
have **sunk** a neglected smack rather than settling her on the mud.

## Verbs

| verb | afforded by | does |
|---|---|---|
| `hail office` | `Helm`/`Boat` (the shipped `hail`) | asks the office for a berth |
| `course <quay>` | the helm | comes in and makes fast (admitted first) |
| `go <her>` / `go ashore` | `BerthExit` on the quay / `AshoreExit` on her deck | aboard and ashore ([ship.md](./ship.md)) |
| `berth <ship> at <quay>` / `berth <ship> roads` | `HarborBook` (peers), transport `harbor` category | the keeper's call |
| `pay office` / `pay book` | `HarborOffice` / `HarborBook` (`Payable`) | pays every bill you answer for |
| `charter [<ship>]` | `CharterBook` (peers) | hires the whole ship until she next berths at a kept quay |
| `look book` | — | the berth book's page |
| `read register` | `RegisterBook` | the register's public page |

## Files

- Transport pack (`packages/content/transport/`):
  `src/idea/Harbor.ts`, `src/thing/HarborBook.ts`,
  `src/location/HarborQuay.ts`, `src/location/HarborOffice.ts`,
  `src/idea/cmd/harbor/BerthController.ts`,
  `content/system/transport/cmd/harbor/berth.yaml`; tests
  `src/__tests__/Harbor.test.ts`.
- Kernel (`packages/server/src/mud/`): `lib/expanse/Berthing.ts`,
  `lib/expanse/Hailable.ts`, `lib/expanse/FrameSited.ts` (`lyingHere`, the
  alongside line, the restart re-wire), `lib/banking/Payable.ts`,
  `platform/thing/CharterBook.ts`,
  `platform/idea/cmd/work/CharterController.ts`,
  `platform/idea/cmd/banking/PayController.ts` (the `Payable` form),
  `platform/idea/cmd/social/HailController.ts`, `platform/idea/ExpanseNode.ts`
  (`hail`), `lib/document/DocumentKinds.ts` (`berth-roll`),
  `platform/idea/api/DocumentLogic.ts` (`saveBerth`, `berthRollOf`).
- Content: `packages/content/terminus/content/world/terminus/harbor/`
  (district, quays, office, line desk, yard, chandler, regulars,
  `idea/harbor.yaml`, `thing/{berth-book,charter-book,register}.yaml`,
  `agent/harbormaster.yaml`), `…/wharfside/idea/harbor-works.yaml` (the
  seat), `packages/content/waponi-woo/content/world/terminus/waponi-woo/`
  (`idea/harbor.yaml`, `thing/berth-book.yaml`,
  `location/{landing,store}.yaml`),
  `packages/content/world-seed/content/stuff/idea/WaterExpanse/greywater/waponi-woo.yaml`
  (`hail:`); `packages/content/terminus/src/__tests__/harbor-content.test.ts`.

## Deferred

| seam | attach point | slate |
|---|---|---|
| wharfage at a private landing (a ship left asleep at the woodyard or falls landing is charged by nobody) | `Harbor.quays`, or a landing's own book | ships-and-harbors-slate § 6 |
| leased berths; the liveaboard's address | a berth line's lease | ships-and-harbors-slate § 8 |
| the roads as an anchorage reached by boat; swinging room | the roads node + `SeaWarren` | ships-and-harbors-slate § 18 |
| the charter as a contract with demurrage; the line's receivables (an unpaid charter blocks only her clearance; a charterer who walks away leaves a receivable nobody collects) | `ship.owed` → the contract substrate, credit's arrears | ships-and-harbors-slate § 17 |
| the harbormaster's posted criteria as a `rate-card` document the committee posts (today they are the `Harbor` row's fields, edited through the CMS) | `Harbor.rates`/`quays`/`exempt` → a document | none (no slate carries it) |
| a dredger; berth depths derived instead of authored | `Harbor.quays[].berthDepthM` | ships-and-harbors-slate § 18 |
| pricing per real hour instead of the world's day | dues, charter and hire rates | the pace slate, open question 4 |
| a harbor reachable by `hail` on a lake (no river line, no node) | `Frame.hailables` | — |

## History

- **Harbors (2026-10-31 → 2026-11-04, MR !511).** Stage A built the port
  (A1 `b66ee4e6f`), the regulars, dues and `pay` (A2 `043d6a4d2`), and the
  charter, the master's helm and Waponi Woo's quay (A3 `ea70e29a5`). The
  way aboard named for her came first (A0 `3c78e91bf`). Review round 1
  reshaped it:
  - `BerthKeeper` was renamed `Berthing` for the act, and the hail moved
    to the hailable side. A sea node names its station
    (`ExpanseNode.hail`), replacing `hailedAt` on the quay (`24d31eeb6`).
  - The berth record moved off the `Harbor` Idea onto the `HarborBook`
    fixture. `Harbor` keeps only its rules and persists nothing
    (`0f11a60ff`).
  - A landing's list of who lies there became a `berth-roll` document
    each ship writes for herself (`1f66507f0`), replacing first the
    register's `lies` field (`ebbb157fe`) and then a `BerthRoll` Idea.
  - The way aboard is called *"the Hesper's gangway"* so that `look
    hesper` on the quay is the ship (`eba27518c`). Its direction is still
    her word.
