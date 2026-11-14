# Ship — a structure that moves, that somebody owns, and that can sink

A `Ship` (`platform/idea/Ship.ts`) is a **`Structure` whose position is a
variable** ([expanse.md § What the frame registers](./expanse.md)). This
doc covers what the harbors build added to her: **kinds** built many
times from one plan, a **warren** of rooms, a minted **identity**, a
**register** and a **flag**, **owners** through her company's cap table, a
**master**, a **gangway** with a lock, a **hull** that wears and leaks,
a **bilge** that is pumped, and **sinking** to a wreck that is still her.
Small craft (`Boat`) share the bilge and are covered at the end.

The harbor's side (the berth book, the harbormaster, dues, the charter)
is [harbor.md](./harbor.md). Her position, the voyage, sight and the
bar's tide window are [expanse.md](./expanse.md) and
[river.md](./river.md). The warren substrate she shares with a house is
[holding.md § The lift](./holding.md).

```
Ship = Bilge(Persistable(Voyaging(Moving(Engaged(Positioned(Durable(Faced(Structure))))))))
```

**Why this shape.** A ship is a structure with a hull: membership, an
entrance and a landing like any building, plus a position, a fabric that
leaks, and title. Her rooms are a **warren one level down**, exactly like
a holding's, so a ship and a house share one substrate for minting keyed
rooms, sleeping whole and saving what must be kept. Nothing about rooms
is invented twice.

⛔ Everything below is on `Ship`, never on `Structure` (a tower does not
moor or sink) and never on `Voyaging` (a boat voyages and does none of
it). *Is it a ship?* is `instanceof Ship`.

## Kinds and authored ships: one model

- **A ship of a KIND** (the smack, `/stuff/idea/Ship/smack`, terminus
  pack): one row every smack is cloned from. ⭐ **Her plan is the kind
  row's own** (`extent: /stuff/location/ship/smack`, `entrance: …/deck`),
  named exactly as an authored ship names hers. The plan rows (deck, cabin,
  hold) are ordinary rows with ordinary `exits:` and each room's own
  `floorM`. They are never served: the location graph attaches nothing for
  a Structure row whose rooms a `warren:` mints, and a kind's membership is
  her warren's (`Structure.memberExtent()` → `''` for her). There is no
  blueprint row ([holding.md § Plans](./holding.md),
  [construction.md](./construction.md)).
- **An AUTHORED ship** (the Hesper, the Ostrander, the Kittiwake, the
  Corrigan): one row is one ship. Her rows are the place, they are served,
  and her warren adopts them.

The hull is `hull: {lengthM, beamM, draftM, depthM?}`. `depthM` is keel to
deck, what she can take in before she is full. Without it,
`hullDepthM()` is 2× her draft.

## Identity and handles

⭐ **A ship of a kind is a minted identity** `<kind row>/<serial>`
(`/stuff/idea/Ship/smack/<serial>`). Her record is `(identity, serial)`
via `PersistableApi.restoreOrSeed`. `Ship.getDurableHandle()` is her
identity (her own row for an authored ship), never `row#key`. Everything
that names her holds it: the register entry's `key`,
`BerthExit.shipHandle`, the berth roll, the wreck record, staging.
`isOfKind()` and `serial()` read it.

- **Standing her up by handle**: `ExpanseApi.standUpCraft(handle)`
  returns the live one, else clones her kind under her identity and
  restores her record (`null`, nothing minted, without a record), else
  (an authored row) `StuffApi.singleton`.
- ⭐ **Her rooms are found through her**, never by a handle lookup (there
  is no `findByDurableHandle`). A reader holding a room **asks the room**:
  `Place.structure()` / `structureNow()` (its warren's `structureOf` for a
  member, else the location graph's attachment by name) and
  `Place.craftHere()` (the place itself when positioned at sea, else its
  structure). `NavigationApi.structureAt(Now)`, `ExpanseApi.craftAt` and
  `leanAtSync` take a row or identity only and **throw** on a `#` handle
  (break loudly).
- Her keywords and her way's word are DERIVED from her name at
  commissioning (`berthWord()`: lowercased, `the` dropped), and are marked
  so in code.

## The warren and her rooms

A `Warren` is a class and a ship is a `Structure`, so she **owns** one:
`platform/idea/ShipWarren` extends the kernel's `FloorplanWarren`, minted
keyed `(warren row, her serial)` (her row for an authored ship) by
`Ship.standWarren()`.

- **A kind's warren** (`/stuff/idea/ShipWarren/smack`, `data: {}`) reads
  her plan off her (`FloorplanWarren.planOf()` → her kind row's `extent`
  and `entrance`) and copies it keyed `(room row, <serial>/<leaf>)`. A way
  inside the extent leads to the same room of HER copy. A way out (her
  `ashore`, her `side`) is as authored. The rooms' zone is the kind's,
  resolved under the room row's folder (`zoneFromFor`).
- **An authored ship's warren** (`/platform/idea/ShipWarren/authored`)
  copies no plan and **adopts** her resident rooms (`adoptResident`). It
  lets them go on destruct and never destructs them.
- `ShipWarren.structureOf(member)` answers *which ship is this room?* for
  every member. A smack's deck row is every smack's, so no path can tell
  two apart.

**Every ship room is a `Compartment`** (`platform/location/Compartment.ts`
= `WaterPlace(Swimmable(Shore(WarrenMember(Location))))`): one kind of
room, instanced keyed for a kind and by its own row for an authored ship.
A deck that cites a `sea` feature stamps itself a shore of whatever sea
she lies on as it wakes (`Compartment.onWarrenWake`, the one wake hook).

⭐ **Only her HOLD keeps a record** (review 2, *persistence is the
exception*). `platform/location/Hold` = `Persistable(Compartment)` is
named on every hold row. Her cargo survives her sleep, a restart and her
going down. Her deck and cabin come up from their rows each wake.

- **`ShipWarren.stow()`**, before she sleeps: what is loose on her plain
  rooms (and in her gear, e.g. stones in the dinghy) goes into her hold,
  and the row gear each plain room is **short of** is recorded
  (`shortOf`, persistent).
- **`fitOut`**, at wake: that many re-laid items are taken back off, so a
  line or flag carried ashore stays gone across every later sleep. A
  plain room's `props:` are her fit-out, not a faucet.
- **Lost by design**: the arrangement and per-item state of her row gear
  (a line moved off the bitts comes back at them, a struck flag is hoisted
  again, a worn line comes back new), and on an authored ship (which never
  sleeps), what is loose on deck at a server restart.
- `Hold.standsUpAlone()` is `false`: her warren stands her rooms up with
  her, never one room by its key (`PersistableApi.standUpKeyed` routes the
  key through the warren).
- Saving her saves her hold: `Ship.dependentPersistables()` = her warren
  (and her minted company), and the warren's = its rooms that keep a
  record.

## Sleep, wake and the restart

- **A ship of a kind sleeps whole** when nobody interactive is anywhere in
  her rooms (sitting in her dinghy on deck counts as aboard) and she is
  not under way (`sleepIfEmpty`, poked by her rooms' population witness).
  `sleep()` brings her lines' ends in silently, stows, does one capture,
  then destructs her warren and herself. She keeps `moored` and `berth`,
  and **her `BerthExit` stays on the landing**, remembering her name, bolt
  and keyway, so the landing's `look`, `lock` and `unlock` still work.
- **Taking that way stands her up** (`ExpanseApi.standUpCraft`), re-ties
  her lines and leads onto her deck. A way taken while she is going to
  sleep waits it out (`Ship.goingToSleep()`) and stands her up again.
- **An authored ship never sleeps**: she stands while her zone does. Under
  way, any ship vetoes eviction.
- **After a restart** a sleeping ship of a kind is attached to nothing
  authored, so the landing's **berth roll** stands her up: the landing's
  first use (`FrameSited.frameSiteStood`) reads its `berth-roll` document
  and stands up every listed ship not loaded. Each re-wires her own way
  (`rewireBerth`) and answers `Voyaging.answerBerthRoll`, striking herself
  if she is no longer made fast there. An authored ship's `berth:` and
  `moored:` re-make her mooring at load (`onRestored`, `seedBornWith`).
  A ship of a kind made fast at a landing sited on no frame is warned
  about, because nothing re-wires there. See [harbor.md § Two
  lists](./harbor.md).

## Buying a ship — the yard's hull book

A ship is sold as **title**, never as a good off a shelf. The yard's
**hull book** (`/system/transport/thing/HullBook` =
`Persistable(ForSale(Good))`, transport pack) lists a **finite, authored**
set of hulls, `hulls: [{slot, kind, priceMinor}]`, plus the `quay:` she is
delivered to. It affords `buy` through the kernel offer protocol
(`ForSaleMixin`, see [retail.md § The offer protocol](./retail.md)), and
it is the venue the price settles to. `buy smack` (or `buy near
stocks`):

1. a closed yard sells nothing; none left → *"the yard has no hull on the
   stocks"*;
2. the register is the one the government over the yard keeps
   (`Government.shipRegister`). None means no sale;
3. her **name** is asked of the buyer and re-asked while the register has
   it, all before money moves. A script (no prompt) cannot name her;
4. ⭐ the slot is **claimed synchronously** before the money awaits, and
   given back if the settle fails, so two buyers cannot both get the last
   hull;
5. the money settles; she is minted `<kind>/<serial>`, seeded, and
   **commissioned** (`Ship.commission(buyer, name, quay, register)`): her
   company is founded with the buyer holding the whole table, she is
   entered in the register as that company's, her gangway is keyed and
   locked with the key to the buyer, the buyer is appointed her master, and
   she is put on the frame at the yard's quay and made fast. The buyer
   hears: *"She is entered in the register of the City of Terminus as the
   Petrel, all sixty-four shares in your name, and you are her master. Her
   gangway is locked; the key is yours."*;
6. the sale is kept in the book's own record (`sold`). Nothing restocks.

The book's page reads *On the stocks: a smack on the near stocks (5000)*,
*Sold off the near stocks: the smack Petrel.*, *Every hull in it is
sold.* Terminus's yard lists two smacks at 5,000 each (about 90 days of a
hand's wage).

## The register and the flag

**A government keeps a book of ships if it chooses to**:
`Government.shipRegister` names a `ShipRegister` row
(`platform/idea/ShipRegister.ts` = `Registrar(Idea)`; Terminus's is
`/platform/idea/ShipRegister/terminus-city`). Entries are `ship` documents
under the government's branch, `/platform/ships/<gov>/<name slug>`
([document-store.md](./document-store.md)):

```
ShipEntry { name, kind, key, flag, registeredAtS, owner }
```

`owner` is the identity path of **her company**. ⭐ The register keeps
**title only**; who holds the company is the company's own table.

- **You file; you do not hold the pen** (the herdbook's rule). Only the
  register writes, through `DocumentApi.saveToRegister`, so no shipowner
  can rewrite whose she is. Every read re-checks the path lies under the
  register's prefix.
- **The slug is the uniqueness**: `slugOf('Kittiwake II') =
  'kittiwake-ii'` (case-insensitive, article dropped). A taken name is a
  taken path, never a scan. The same name under another flag is another
  ship.
- **The one write** is `enter`, gated `FromMixin(Voyaging, where: the
  caller's handle is the entry's key)`: the ship herself, about herself.
  `commission` throws if she already names a register.
- ⭐ **A ship knows her register** (`Ship.shipRegister`,
  `registeredName`, both authorable). `registerEntry()` is one keyed read
  under her name, checked to name her handle. Nothing walks every
  register, and there is no `ShipRegisterApi`.
- The authored ships' entries ship in the platform pack
  (`content/ships/terminus-city/{hesper,ostrander,kittiwake,corrigan}.yaml`).
- `read register` on the harbor office desk (`RegisterBook`,
  `Marked(Good)`, `registerPrefix: /platform/ships/terminus-city` — the
  same view class as paper's ledger; a ship entry's line is rendered by
  its kind) derives its page each read: name, kind, flag and owner,
  and beside it who holds that company (*"…: owned by the Petrel's
  company, held Ann 64/64."*). The register is public, so false colors can
  be caught by anyone.

**The flag** is a government's brand (`Brand.government`, e.g.
`/stuff/idea/corpo/Brand/flag-terminus-city`) on a `Flag`
(`Branded(Good)`) flown `on` a `Flagstaff` (a `Placing` fixture taking one
flag; `put flag on flagstaff`, `get flag`). `Ship.flownFlag()` is the
flag on her deck's flagstaff, and her `look` says *"She flies the colors
of the City of Terminus."* ⚠ That is only a **claim**. Her **flag state**
is her register's (`Ship.flagState()`), never the cloth's.

**The law aboard** (`GovernmentApi.subjectTo`): aboard a ship is asked
**first** (her rooms are in her own zone, which no address covers, so
the question is *is this a ship*, never *is the chain empty*). In harbor
(made fast at a landing, or on a river within 2 km of one) the port's
chain comes first, then her flag state's. At sea it is the flag state's
alone. A stateless ship at sea is under nobody's law.

## Owners, the managing owner, the master

⭐ **Her company owns her** (review 1, `362ec52aa`). Owners are a **cap
table**: `CapTableMixin` on a `PersistentBusiness` (`Persistable(
CapTable(Business))`), whose appointing authority is `{kind: owners}`.
That is the institutions slate's primitive, and ships are its first
consumer ([employment.md § The owners authority](./employment.md)).

- A smack's company is minted at commissioning from her kind's `house:`
  seed (`/stuff/idea/Business/ship-company`, `capTable: {units: 64, rows:
  []}`) at `<seed>/<serial>`, with the buyer as the whole row. It is kept
  `(seed, serial)` and listed among her dependents. An authored ship's
  company is her `house:` row (the Hesper's outfit, the regulars' companies
  and the Greywater Line's office at 64/64; the Ostrander's held by
  `/world/terminus/budget`).
- `Ship.company()`, `managingOwner()` (a holder of **more than half** the
  units, else nobody), `shareHolders()`, and `refreshOwners()` (cached for
  the synchronous `answersFor`).
- **`title transfer <n> shares of <ship> from <holder> for <price>`** is run by
  the **buyer**: the holder confirms, and `CapTable.sellUnits` claims the
  units, settles the money and gives the units back if the money fails.
  **`title transfer <n> shares of <ship> to <person>`** is the holder's gift
  (`giveUnits`). The verb finds her through the register kept by the
  government whose law runs where the act is done (a bill of sale is
  lodged under her flag), then `standUpCraft`, then `company()`, checked
  against the entry's `owner`. Under a `consent` term every other holder
  is asked.
- ⭐ A unit is a vote in the company, never standing in the polity (lens 7).

**The master** is the company's `master` seat. The buyer of a smack holds
all three roles: owner, managing owner, master. Whoever holds a majority
appoints her master with the shipped `house appoint`. **`Ship.mayBeSteeredBy`**
(a `Voyaging` hook; a boat answers anyone) is her charterer while the
charter runs, or whoever holds her `master` seat. `course` and `anchor`
refuse others: *"Only her master takes the helm — charter her, or be
appointed to command her."* **`Ship.answersFor(who)`** (who pays her dues
and bills) is the master, the charterer, the managing owner, or whoever a
bill of hers names.

## The gangway

Her gangway is a door with a key. She carries `gangwayKeyway` and
`gangwayLocked` (persistent). A bought ship gets a minted keyway, is
locked, and the key goes to her buyer. Further keys are physical `Key`s
handed over (no key-cutting verb, as for a householder). Authored ships
author an empty keyway, unlocked.

- **`BerthExit`** (on the landing, `Lockable(DeferredDestinationExit)`,
  row `/platform/idea/exits/berth`): its direction is her word (`go
  hesper`, *Obvious exits: hesper*), its name is *"the Hesper's gangway"*
  (so `look hesper` on the quay is the ship and `look gangway` reads the
  way's own sentence, *"A gangway runs up to the Hesper's deck."*), and its
  keywords are `gangway` + her word. It refuses a locked gangway to anyone
  without the key. It mirrors her bolt and keyway so it answers while she
  sleeps. `alongsideNow()` is her while she stands and lies at that
  landing, else `null`.
- **`AshoreExit`** (on her deck): its far side is her `getBerth()` first,
  then the landing within 60 m on a river or the node's landing at sea.
  Between nodes it refuses in words. It composes `Lockable` too (`lock
  ashore` from the deck), but ⛔ **leaving never refuses**: its gate never
  reads the bolt.
- Neither way is ever mapped (`isMapped() → false`): *maps never name a
  ship*.

## Alongside

Mooring is `course <quay>`; the berth side (hail, admission, dues, the
book) is [harbor.md § Berthing](./harbor.md). Her side:

- ⭐ **Her lines are cordage's ropes.** A rope on her deck kept **at her
  bitts** (`/platform/thing/bitts`, a `Bollard`: placed `on` it or tied
  round it, long enough, ≥ max(2 m, 2 × deck height)) is one of her
  mooring lines (`linesAboard`). A rope loose on deck is not.
  `put rope on bitts` or `tie rope to bitts` makes any rope one. Her
  rows prop `{template: /platform/thing/mooring-line, count: 2, on: bitts}`.
- **Made fast** (`comeAlongside`): each line's end is minted round a
  `Bollard` on the landing with a **clove hitch** through `Rope.tieTo`
  (only the structure the rope's room belongs to may tie across); the
  rope stays aboard. Then the `BerthExit` goes on, her own line goes on
  the landing's berth roll, and a kept quay is told `madeFast`. With no
  line, a course for a landing is refused before it is laid (*"She has no
  lines to make fast."*). With no bollard ashore she lies off.
- ⭐ **Moored is a read** (`fastLines()`: the ropes she tied whose end is
  still round a bollard). `untie` of the end on the quay, or of the rope
  aboard, tells her (`Line.endUntied` → `Structure.onLineLetGo`). When her
  last line comes off she is adrift and the way is removed. Any course
  from a moored ship casts off first.
- She persists only `berth` and `moored`. Rope ends are runtime, re-tied
  when she loads.

## The hull: coat, seams, leak

⭐ **A ship's hull is hers, the same fabric model as a house's** (paint,
merged into harbors). `Ship` composes `DurableMixin(FacedMixin(...))`:

- **Coats** (tar, paint) weather **wherever she is**, at sea most of all
  ([painting.md](./painting.md), [faces.md](./faces.md)).
  `persistsCoats()` is `true` for a ship.
- Under the coat her **seams** are a `Durable` condition. Once the coat is
  gone they wear at the house's rate, `structure.weather.daysToWorn`
  (default 45 game days from sound to worn). Ships author no coat, so they
  start bare. Players read the house's words: *sound · worn · badly worn ·
  broken*.
- **The leak** reads her condition in five **seam bands** (≥0.9 sound,
  ≥0.7 weathered, ≥0.5 worn, ≥0.25 shabby, else dilapidated) as litres a
  second per meter of length (`ship.leak.<band>`), plus `ship.leak.holed`
  for a hole. `getCondition()` / `reconcileWeather()` integrate the bilge
  **first**, so the leak is integrated band by band along the hull's line
  and the bilge stamp never trails it.
- **`maintain`** aboard (or named from the quay) with a **caulker's kit**
  (Hobb's, 12) caulks her sound, charges the water already in at the old
  rate, and credits **seamanship**. The residence pack's
  `MaintainController` carries both word sets (a house is puttied, a hull
  is caulked). `analyze rooms` aboard reads her like a building (the shell line), and
`analyze condition <ship>` from the quay says why her hull is as it is.

## The bilge, the pump well, `measure bilge`

`lib/expanse/Bilge.ts` (`BilgeMixin`, shared with `Boat`) holds the
litres and the stamp (`bilgeAt`, `integrateBilge`, `takeBilge`,
`resetBilge`, `bilgeCapacityL`). Each composer says where the water comes
from (`leakedBetween`). A ship's is `null` while her warren is down, so
the stamp waits. Her capacity is her hull's box to the deck (`L × B × 0.7
× depthM`).

- **`BilgeWell`** (`platform/thing/BilgeWell.ts`, a `Thing` fixture on her
  deck, row `/stuff/thing/bilge-well` with `/stuff/thing/bilge-pump` set in
  it) answers `PumpSource` + `LiftSource` off her. `pump well` is the
  shipped `pump` verb ([pump.md](./pump.md)): one stroke a spell, the
  level falling by each stroke and rising between. The water goes over her
  side (`Ship.pumpOut`). Each stroke credits seamanship. The pump's leather
  packing wears like any pump's.
- **`measure bilge`** is a kernel reading row (`/platform/idea/reading/bilge`,
  `BilgeReading`, instrument capability `well-sounding`; the
  `/platform/thing/sounding-rod`), in words for everyone (*"There is water
  knee-deep in her bilge. It is rising steadily; she is making water at her
  seams."*), more detail by seamanship band.
- **Seamanship** (`platform/idea/Discipline/seamanship`, `requires:
  [ropework]`) is credited at making fast (`comeAlongside`), casting off
  from a course, an arrival with no landing, each `pump well` stroke,
  `measure bilge`, and `maintain` on her hull. ⭐ Competence buys
  **information, never outcomes**: a better seaman reads the bilge in more
  detail (the rate, the rod height, pump against leak). Nobody pumps
  faster or leaks less for it.

## Flooding, settling, sinking

⭐ **A compartment asks its ship.** Every water read on a `Compartment`
(`levelNow`, `depthNow`, `stateNow`, `mediumFor`) is
`Ship.floodDepthAt(room)` = her flood height (bilge spread over her hull's
box) less the room's `floorM` (authored per room; else her deck at her
hull's depth and any other room on her keel). Her rooms flood from the
lowest up by arithmetic, and nothing is stored on a room. ⚠ **Dry is
dry**: a dry compartment's `depthNow()` is 0 and `bandsNow()` is always
empty, and a person is in the AIR unless the water is over their head.
*Is this the water* is `depthNow() > 0` everywhere (there is no
`isWaterNow`).

`Ship.noteFlood()` decides after every reconcile, every stroke, at
stand-up, and when her flood watch (a game-time timer re-armed at each
band boundary) comes due. Nothing is rolled:

- **Settled, not sunk**: full to her deck where the water under her
  (`bottomDepthM()`: a kept berth's `berthDepthM`, else the frame's
  `depthUnder`) is shallower than her hull. *"She has settled onto the
  mud, the water standing in her to her deck."* She refuses the helm
  until pumped. She floats again once her draft plus the water in her
  clears the bottom, or on a drying berth once pumped to her floors
  (0.15 m).
- **Settling**: full to her deck in deeper water. *"She is settling."*
  ⭐ The window is **per person, in acts** (*no window shall be shorter
  than the act that answers it*). It is per person because a shared count
  would let five people aboard close it in one command each, so nobody
  would get the time to act: each interactive person aboard has
  `ship.settlingActs` (6) commands of their own, counted through
  `CommandDispatchedEvent` (it carries the giver). A person who leaves is
  out of the count. She goes down when **everyone still aboard has used
  theirs**, or after `ship.settlingRealS` (600) real seconds (the backstop
  for an idle body), or at once with nobody aboard. Pumped back below her
  deck, *"she is holding"* and the window closes.
- **Sunk** (`Ship.sink()`): she anchors where she is, casts off for good,
  records `wreck: {atS, depthM}`, and is full. Everyone aboard is told
  *"She goes down under you."* Her title, register entry, cargo and rooms
  are untouched. **The wreck is the ship.**

## The wreck

The water keeps a surface over her and a way down
(`Frame.stopOver(ship)`, the water pack): a `SeaStop` at sea (a keyed stop
with a column from the depth), or a `ReachStop` on a river (whose floor is
the bed). Both compose `WreckSiteMixin`, which persists `wrecks` and wires
the ways whenever the stop or her deck stands:

- when the stop's **bed band buds** (`WaterPlace.onBandBudded`), a way
  **named for her** (her word, `hesper`, by the berth rule) leads from the
  bed onto her deck;
- her deck's `up` leads back to the bed (at sea) or the stop (on a river),
  so someone still aboard swims out and up.

Sunk, every room reads the bottom's depth less its floor, so a wreck's
plain rooms are under the sea the moment they stand. Between two
under-water rooms a dry way admits a water mode (`Exit.floodedFor`), so a
diver `swim down`s her companionway. No chart knows where she lies until
somebody charts her (Deferred).

## Aboard: `route`, `locate`, `look`

- **`route`** aboard any ship plans from where she **lies**. Moored, the
  plan starts at her landing with `ashore from <her>` as its first leg.
  Not made fast, it is refused naming `course` (*under way*). A ship's
  rooms are never an origin; the map cannot claim a way that left with
  her.
- **`locate`**: at sea, the reckoning; otherwise *"aboard the Petrel,
  made fast alongside the yard quay."* or *"aboard the Petrel, under
  way."*
- `Compartment.isSeenAcrossExit()` (her deck while afloat) and
  `BerthExit.peekDestination()` (asks her, `watchRoomNow`) feed MQL's
  steps between places; `look` does not swap targets (the one resolution
  rule, [mql.md](./mql.md)).

## Boats

A `Boat` (`/system/transport/thing/Boat`, transport pack) is a small
craft: a **thing** in a water room, not a place ([river.md](./river.md)).
The harbors build gave it the ship's water and the ship's market,
scaled down. It lives in this doc because the bilge is one substrate:

- **Bilge and leak**: `Boat` composes `BilgeMixin` and `FacedMixin` (her
  outside takes tar). She leaks from her **state**, never the calendar:
  holes (`boat.leak.holed` 0.3 L/s each, a `holed` flaw) and a coat that
  no longer holds water (`boat.leak.seep` 0.02), integrated only while
  afloat. She goes down with what is in her (`room.sink(boat)`), or swamps
  where she lies at a columnless stop.
- **`bail`** is the bucket over any `LiftSource` ([pump.md § The
  bucket](./pump.md)). Unnamed, it takes the boat you sit in. A bucketful
  is 10 L a 20 s spell by hand (beats one hole), and goes *"out over her
  side"*.
- **Swamping by load**: her freeboard falls with what is in her
  (`freeboardNowM`), and `reconcileFloat()` runs when her load changes or
  she is moved. Freeboard ≤ 0 swamps her (*"She is loaded past her
  freeboard."*). Otherwise the shipped wave rule applies, afloat only.
- **Owned**: a boat is an ordinary good, **sold where she lies**. The
  put-in counter prices it (rowboat 160, canoe 110) with no stock line, and
  `buy rowboat` transfers title to a house boat lying loose in the
  counter's room without moving her (*"It is yours where it lies."*). The
  put-in is a `PersistentShore` so a sold boat does not come back on the
  next restage. See [retail.md](./retail.md).
- **Hired**: `rent <boat> [for <n> days] [at <book>]` at the shed's
  **hire book** (`platform/thing/HireBook`, authored `rates` per kind).
  The hirer pays up front and the boat is put in their **custody**, never
  title (`HirableMixin.hireTo`). ⭐ The boat's hire is the only record;
  the book persists nothing and reads which boats are out, and overdue,
  off the boats. The hire ends when she is put down on the landing she was
  let from (`Boat.onMoved` → `endHire`). Overdue is a fact on the page,
  never a penalty.
- Every boat kind props a bow line (`/platform/thing/bow-line`).

## Staging (tests and drives)

`TestHarnessApi.stageShip(handle, spec)` (`@TestOnly`), the `testharness
ship` subcommand, `POST /auth/test-ship`, and the wire helper
`stageShip` (`packages/wire/src/harness/world.ts`):

```
testharness ship <ship> [at node:<path> | at <frame> <cite>] [hull <band>]
                 [bilge <L>] [slept <days>] [flag <government>] [settling]
```

The ship is named by row, identity, or her registered name (`flag`
picks the register, Terminus by default). `hull <band>` stages the band
she READS (`Ship.stageHull`, mid-band). `slept <days>` moves the bilge
stamp back and integrates over the hull's line extrapolated backward.
`bilge <L>` sets the litres (`stageBilge`). `settling` opens the window.

## Settings (grain)

`packages/content/platform/content/settings/ship.yaml`:

| key | value | meaning |
|---|---|---|
| `ship.leak.sound/weathered/worn/shabby/dilapidated` | 0 / 0.0005 / 0.004 / 0.02 / 0.08 | L/s per meter of length by seam band |
| `ship.leak.holed` | 0.25 | a hole on top of the seams, L/s/m |
| `ship.settlingActs` | 6 | commands each person aboard has once she settles |
| `ship.settlingRealS` | 600 | the backstop, real seconds |
| `boat.leak.holed` / `boat.leak.seep` | 0.3 / 0.02 | a boat's leak, L/s |
| `structure.weather.daysToWorn` (`structure.yaml`) | 45 | the house's and the hull's dial |

⭐ Sized so *nothing fails over a weekend away* and *a neglected season
settles her*: a sound smack left alone has about 5,800 L in her at day 24,
about 16,300 L at day 30, and reaches her deck about day 36. The Hesper
reaches hers about day 48. One bilge pump (0.6 L/s) gains on a worn hull
and is outrun by a dilapidated one. The arithmetic is in the settings
file's header.

## Files

- Kernel (`packages/server/src/mud/`): `platform/idea/{Ship,ShipWarren,ShipRegister,BerthExit,AshoreExit,PersistentBusiness}.ts`;
  `platform/location/{Compartment,Hold,PersistentShore}.ts`;
  `platform/thing/{Bollard,BilgeWell,CharterBook,HireBook,Flag,Flagstaff,RegisterBook}.ts`;
  `platform/idea/reading/BilgeReading.ts`; `lib/expanse/{Voyaging,Bilge,Hailable,Berthing,FrameSited}.ts`;
  `lib/location/{FloorplanWarren,Floorplan}.ts`; `lib/persistence/KeyedHosts.ts`;
  `lib/employment/CapTable.ts`; `lib/retail/{ForSale,Hirable}.ts`;
  `lib/structure/Structure.ts` (`memberExtent`, `onLineLetGo`);
  `lib/stuff/Place.ts` (`structure`, `structureNow`, `craftHere`);
  `api/expanse.ts` (`standUpCraft`); `platform/idea/api/TestHarnessLogic.ts` (`stageShip`);
  controllers `cmd/work/CharterController`, `cmd/retail/RentController`,
  `cmd/device/BailController`, `cmd/civics/TitleTransferController` (the shares
  form), `cmd/system/TestHarnessShipController` (`ship`), the `course`/`anchor`
  helm gate, `cmd/perception/LocateController`.
- Transport pack: `src/thing/{HullBook,Boat}.ts`; tests `HullBook.test`,
  `Boat.float.test`, `Boat.leak.test`.
- Water pack: `src/lib/WreckSite.ts`, `src/location/{SeaStop,ReachStop}.ts`.
- Residence pack: `src/idea/cmd/crafting/MaintainController.ts` (the hull's
  words).
- Content: the smack kind (`packages/content/terminus/content/stuff/idea/Ship/smack.yaml`,
  `…/ShipWarren/smack.yaml`, `…/stuff/location/ship/smack{,/deck,/cabin,/hold}.yaml`);
  `packages/content/platform/content/platform/idea/ShipWarren/authored.yaml`;
  the register (`…/platform/idea/ShipRegister/terminus-city.yaml`,
  `content/ships/terminus-city/*.yaml`, `Government/terminus-city.yaml`
  `shipRegister:`); the yard's hull book
  (`…/world/terminus/harbor/yard/thing/hull-book.yaml`); the put-in
  (`packages/content/rejection/content/world/terminus/rejection/put-in/`);
  `settings/ship.yaml`.

## Deferred

| seam | attach point | slate |
|---|---|---|
| re-flagging; a ship moved to a second register (`commission` refuses a ship that already names one) | `Ship.shipRegister` + a register-to-register transfer | ships-and-harbors-slate § 3 |
| a wreck on the map (`charted`) | a `Chart` entry naming a `SeaStop`'s handle; a diver's `chart` act | ships-and-harbors-slate § 7 |
| salvage, raising a wreck, the careening hard | `Ship.wreck` | ships-and-harbors-slate § 7 |
| crews, the watch, mates at the helm | `mayBeSteeredBy` reading more seats | ships-and-harbors-slate § 15 |
| a shipwright laying down a new keel to restock the hull book (it is finite and nothing refills it) | `HullBook.hulls` | none (no slate carries it) |
| the boatwright restocking the put-in by work | the put-in counter's `prices` + an act placing a boat | ships-and-harbors-slate § 9 |
| the hire's overdue consequence (`HireBook.overdueHires` is read-only; nothing enforces a late return) | `Hirable.hiredUntilS` | none (no slate carries it) |
| a let's book shared with a lease (custody for a term) | `HireBook` and the Seznick lease | — |
| endorsements, forms, primary issue, dividends on a cap table | `CapTableMixin` | institutions-slate |
| wet rope rotting in a flooded hold; a river wreck as an obstruction; a ship sunk at an unkept landing telling the landing | `Ship.sink`, `WreckSite` | ships-and-harbors-slate § 7 |
| logging off aboard a smack lands you at the realm's default start (`admitFor` knows no ships) | `CirculationWarren.admitFor` | — |

Open from the drive (2026-11-04): from the sea `go <ship>` names `climb`,
which a swimmer has no verb for (climb/swim slate); at sea the smack's deck
still lists `ashore` among its exits.

## History

- **Harbors (2026-10-31 → 2026-11-04, MR !511).** Stage B built ship kinds
  and the warren (B0 `4f42b700d`, B1 `7582615d3`), the register, the flag
  and the yard (B2 `1e6d6ea6e`), owned and hired boats (B3 `a2922de5c`),
  the hull clock, the pump well and seamanship (B4 `461db4993`), and
  flooding, sinking and the wreck (B5 `dde366714`). Review round 1
  reshaped most of it:
  - **Mooring.** The `MooringLine` mixin was folded into cordage's
    `Rope.tieTo` and the bitts (`88e6d3045`).
  - **One bilge.** `Bailable` was replaced by `bail` over any pump
    `LiftSource`, with one `BilgeMixin` for ship and boat (`c60944860`).
  - **Register.** `ShipRegisterApi` went: a ship knows her register
    (`ebbb157fe`).
  - **Identity.** A ship is a minted identity whose rooms are found through
    her; `findByDurableHandle` went (`a2f34a285`).
  - **Persistence.** Only the hold persists (`e6b86cfbd`), saved as her
    dependent (`e3141e2d3`). There is one wake hook (`abdae7e98`), and
    `standUpKeyed` resolves by key (`3c48a3cc9`).
  - **Plans.** The `floorplan:` schema, then blueprint rows, went: the
    plan lives on the ship kind's row (`b00f8d45c`, `a20704a98`).
  - **Market.** Ships are sold as title off the hull book (`af492d81d`),
    behind one offer protocol (`d898a4a15`). Boats are sold where they lie
    (`0b7c75e60`). A hire is a let with one record, the boat's
    (`f3305cb19`, `dc5d71a94`).
  - **Owners.** Owners are a cap table and the register keeps title only
    (`362ec52aa`).
  - **No `isWaterNow`.** It is `depthNow() > 0` (`d5e2c84f9`).
  - **The paint merge** (`5cfd1a5db`) retired the warren's
    `WeatheringMixin` for a hull that is `Durable` under `Faced`.
