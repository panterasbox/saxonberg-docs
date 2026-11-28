# Carrying

The carrying subsystem answers **where a thing goes on a body, and what
the body knows about the things it carries**. ⭐⭐ **`ContainmentApi.move(item,
carrier)` is the one call, and a carrier ROUTES what arrives** (size and
carrying W8; owner, 2026-11-13): a hand, a pocket, a pouch on the belt —
else it reaches the body and the body puts it in the bag at its side or
the pack on its back — else it reaches the body and is set down at its
feet, with a line. Everything on a body is **held**, **worn**, or
**placed** on something worn. Nothing sits loose.

That one invariant is what makes hands mean something. A full hand is a
choice that costs you — what you hold is what you can use now, what is
in a pocket or on the belt is a moment away, what is in the pack is
packed — and the same body knows where each thing lives and how long it
takes to get it out in a fight.

An author needs one call: **`move`**. No caller ever chooses how a
carrier takes a thing (owner: *"I don't want people to have to care what
they're moving to determine which method to use"*). ⭐⭐ **There is no way
around the routing** (owner, 2026-11-15: `directMove` retired): a purchase,
a tip, a gift, a store withdrawal, an untie or a reclaim that the recipient
cannot carry lands at their feet with the routing's line — honest, nothing
lost. To **ask** without moving, `carrier.wouldSetDown(item)` reads the
same resolution dry.

The tape measure every placement asks is [size.md](./size.md). The
weight is [encumbrance.md](./encumbrance.md). The slots are
[slot.md](./slot.md) and [embodiment.md](./embodiment.md).

## The Cast

| Piece | Where | What it is |
|---|---|---|
| **the hook** — `arrivalFor(item, how?)` | `ContainerMixin` (`lib/spatial/Container.ts`), beside `canAddContainable` | where an arrival goes IN this container: `contents` (the default, every ordinary container) · `seat` (slots) · `pair` (a worn host's region) · `refuse`. Every kind is in the destination — a bag is not the body |
| **the chokepoint** | `ContainmentLogic.move` | a destination that is an agent is asked `arrivalFor`; `move` performs `seat` / `pair` / `contents`. A `refuse` on a **carrier** goes to the carrier's routing (`Carrying.routeArrival`, gated to `ContainmentLogic`); on a non-carrier it **throws** a `ContainmentError` (`reason`, `facts`, a message naming the body, the item, the hands and what was tried) |
| **the routing** — `Carrying.routeArrival` | `lib/slot/Carrying.ts` | decides where it goes on BEFORE it lands (no bag and nowhere to stand → throws, nothing moved); then a TRUE move onto the body (admitted for the moment it is loose — its witnesses fire), then a second true move: into the first worn or carried bag it fits, else down at the body's enclosing place with the line (*Your hands are full — you set the cup down.*; peers: *Mara sets the cup down.*) |
| **the question** — `Carrying.wouldSetDown(item)` | `lib/slot/Carrying.ts` | the routing's answer read dry, by the same resolution: `null` when `move` keeps it (on the body or in a bag it carries), else the `ArrivalRefusal` it would set it down with. Never moves. Its callers: `get` (refuses a pick-up that would only be set back down) and a staged body's `props:` (an author diagnostic). ⛔ Not a way around the routing — `directMove` was retired (owner, 2026-11-15) and `lint:carry-moves` holds both its name and the `isCarrying(x) ? … : move(…, x)` branch at 0 |
| **the invariant** — nothing loose | `Creature.arrivalFor` (every body) | admits only what arrives placed, an agent, a thing being seated (`Slotted.isSeating`), or one the body is admitting (`Creature.admitWhile`, gated `AnyOf(SelfOnly, FromModule(PersistableLogic))` — the restore, and the routing's first move). Else `refuse` `nowhere-to-carry`. **Fails closed**: no species row loaded is no escape |
| **the seating mark** | `Slotted.seatInSlot` / `seatInSlots` | contents first, then the slots, under a mark (`try/finally`, a failed claim rolled back out) |
| **the capability** — `CarryingMixin` | `lib/slot/Carrying.ts`, composed on `Character`, `DraftAnimal`, `KeptAnimal` and ranching's `WorkingAnimal` | overrides `arrivalFor` with D11's order; the routing and its dry question; tiers, `stow`, readying into a free hand, letting go — **12** interface members (`freeHands · tierOf · isHonest · ready · stow · bringToHand · releaseHeld · wieldPlanFor · gearNoise · returnBorrowed · routeArrival · wouldSetDown`) plus the `arrivalFor` override |
| the hands | `Attired` (`handSlots`, `heldThings`, `wears`) | the slot substrate's fact — any body's hands by its plan |
| the worn bulk | `Attired.wornBulk()` | what is worn on `back`/`torso` and hung at the flanks — the envelope's depth and width ([size.md](./size.md)); moved off `Creature` at finalize (owner, 2026-11-16) |
| `CARRYING_DEFAULTS` | top of `Carrying.ts` | the grip dials (one-hand fraction, two-hand length); the awkwardness dials are `LOAD_BEARING_DEFAULTS`' |
| `carry.order` | `Carrying`'s `static settings` | `hands-first` (default) · `pockets-first` |
| pockets | `PlacingMixin` named regions over the `/platform/placements/in` document, on a `Garment` that is a `Container` with no inside of its own | a region of a garment, never an object; the pocketed thing is the garment's contents (W9) |
| the belt | `PlacingMixin`'s `from` member + its hanging length | a `Garment` row with `placements: [from]` — no belt class |
| worn and hung containers | `Pack` (`Wearable(Slottable(Vessel))`, composes `Placing` and `Sealable`) | a satchel, a rucksack, a belt pouch, a pannier, a holster (`shapedTo:`) |
| the words | `CommandController.carryRefusalWords` · `GetController`'s private `drawWords` | the body reports facts; the verb words them |

**Who carries.** `Carrying` composes only where a body carries: a person
(`Character` → every Avatar, NPC, Extra, Cast), a draft animal (a mule's
pack saddle — panniers hang), a kept animal (the dog's mouth is a hand;
the cat answers `nowhere-to-carry` *by its plan*), a working animal. A
`Beast`, ranching's `Livestock`, a `Fish`, a bare `Creature` do **not**:
they are not routed: `move(item, <any of these>)` throws unless the
arrival is placed, seated, an agent or admitted (grounded 2026-11-13: no
shipped caller moves a loose thing onto a non-carrier — a catch goes to
the fisher, a lead hangs from the halter, tack is seated) — a hog's halter is *seated* (`equip`), and it is
led by it. A `Corpse` is a Thing. Narrow with `MixinApi.isCarrying(x)`
(`Creature & Carrying`). No Api.

## ⭐⭐ `move` — the one way onto a body

```typescript
ContainmentApi.move(item, carrier);           // routed: on it, its bag, or at its feet
carrier.wouldSetDown(item);                   // asks only: null (kept) or the set-down's refusal
```

`Carrying.arrivalFor` is one resolution (`resolveReceive(item)`), which the
routing and `wouldSetDown` both read — so the two can never disagree. In
order, after the
body's admissions (`super.arrivalFor`):

1. **Already in a hand** — `contents`, nothing to do.
2. **A handless body handed its tack** puts it on (`seat` its claim) —
   `put pack saddle in mule`.
3. **The hang rule.** A worn container that claims no slot of this body
   (a belt pouch, the medic's pouch, a holster) **arriving from off the
   body** is hung `from` the first worn host that offers it with room
   (`pair`) — *you clip the holster to your belt*.
4. **A stack meets its like** in a hand, a pocket or a ready container
   and merges — it takes no hand (a like in a bag is the routing's).
5. **`carry.order`** between a free hand (or two — the grip) and the
   first **carry place** that fits — a worn pocket (`pair`), and for the
   routing a hung or worn container, the pack on the back — never into
   itself.
6. Else `refuse`: `too-heavy` · `hands-on-cart` · `cannot-grip` ·
   `nowhere-to-carry` — with facts: what the hands hold, what was tried, a
   held thing that **would** go away and where (`away`), and whether the
   thing takes both hands while one is free (`needsBoth`).

⭐⭐ **Every step is a TRUE move — nothing is retargeted** (owner,
2026-11-13). When the routing's choice is a bag, or nothing, the thing
first reaches the body (`move(item, body)` happens; the body's
`onContainableAdded` fires), then the body moves it on — into the bag
(the bag's fit check and witnesses), or down at its enclosing place, with
the line. Observers see *it reached Mara; she put it in her pack / set it
down*. The order is pinned by `ContainmentLogic.arrival.test.ts`. Only a
carrier with no bag and nowhere to stand throws (decided before anything
moves). A non-carrier (a fish, a canary) is not routed and throws: a
fish's inside is a stomach, not a pocket. A key moved out of its pocket
onto its owner asks the hook like any arrival and goes to a hand.

⭐ **The refusal is the tutorial**, worded by the verb's controller
(`carryRefusalWords`) from `wouldSetDown`'s answer (`get`): *"Your hands are full —
the mug and the lantern — and the map won't fit in your pockets. Put
something away (`put mug in left pocket`) or set it down (`drop
lantern`)."*

### The idioms — one call

There is exactly one movement verb (owner: *"every time you invent a
synonym for `move()` you're making it harder for everyone else in the
game"*), and it routes for a carrier. Nothing goes around it (owner,
2026-11-15 — the refusing form and any new name for it were both
rejected):

| idiom | written as | used by |
|---|---|---|
| **everything** | `ContainmentApi.move(item, dest)` | made things (crafting's `landOutput`, the taps, the harvests, a catch, the cuts, the casts, a drink made for you, a minted card / key / receipt / coins the ledger has moved), staging (`hand`, a kit), restores, an NPC arming itself, `unequip` |
| **handed** — the money, then the goods | `ContainmentApi.move(goods, carrier)` after the write — routed; with nowhere for it, set down at their feet with the line. ⚠ The old *"refused before money moves"* requirement (D-W7a) is **retired** (owner, 2026-11-15) | `buy` (the shelf's `handOver` after `settleSale`), `give`, `put X in <person>`, a rack or shelf reclaim, a store withdrawal, untying a line, a tip jar, a lead tied to a carrier, `equip` out of a wardrobe, the daypack at intake |
| **carried, or refused** | `carrier.wouldSetDown(item)` first, then `move` | `get` (and `get X from <body>`): a pick-up the routing would only set back down is refused before anything moves (off a wall, out of a slot) — `get` carries; a set-down is `drop`'s or `put`'s act |
| **staged** | `wouldSetDown` then `move` | `props:` onto a body — what the routing would set down is a diagnostic, not placed |
| **wear it** | `body.seatInSlots(garment, claim)` | outfits, loadouts; a bag that claims no slot is `move`d and hung |

`seatInSlots` stays its own entry point: it is `Slotted`'s primitive (a
rack, a battery bay, `put`'s slot route, the chokepoint's own `seat`, and
`unequip`'s re-seat into the hand a refused `stow` left) — a loadout that
cannot seat should throw loudly, which `seatInSlots` does.

### No restore-time placement (cut at finalize)

There is no overflow. The invariant holds at every `move`, so a snapshot
holds nothing loose, and a restore puts back exactly what was captured
(`PersistableLogic.restoreItem` → `admitWhile`). The pre-flip restore —
`placeLoose`, the residence overflow, `placesLooseOnRestore` and the
player's placement at `enter` — was **cut** (owner, finalize 2026-11-16:
*no migrations* — no record from before the flip exists to place). A
restored body that nonetheless finds something loose on itself reports
it as a diagnostic (`Creature.onRestored`) and moves nothing.

`Carrying.isHonest(item)` is the walk the invariant means: a slot
occupant, or placed on a host that is itself honest on the body, or
inside such a thing.

## Hands and grip

The hands are `Attired`'s (W8) — the slot substrate's fact, so any body
answers them by its plan; the grip and the free hands are the carrier's.

- **`handSlots()`** (`Attired`) — the body plan's slots whose `bodyPart` serves
  `manipulation`: the biped's `hand:left`/`hand:right`, the `canid` plan's
  `mouth` (a dog carries a stick, a lead's end — one thing). A horse has
  none.
- **`freeHands()`** — hand slots that are empty and can grip
  (`Vitals.canGrip`), **less a hitched cart's `handedness`**.
- **`heldThings()`** (`Attired`) — what the hands hold. `held` means a hand slot,
  everywhere: the load gauge, MQL's `:held`, `wornStack` (which skips
  hands — a coat in your hand is held, not worn), `equip`.
- **A hand takes any good** — `Good` and `Vessel` are slottable, and a
  carrier's hand slot admits any slottable thing whatever its `accepts`
  ([slot.md](./slot.md)). The grip decides.

### ⭐ Two hands by weight, two hands by use

The grip (the carrier's private `handsFor`, read through
`wieldPlanFor`):

- `null` — heavier than the strain ceiling (`too-heavy`);
- `2` **by weight** — mass over `ONE_HAND_FRACTION` (0.3) of the holder's
  carry capacity, or longest side over `TWO_HAND_LENGTH` (0.6) of its
  stature. Derived, so a gnome needs both hands where an ogre needs one;
  no strength stat is added;
- `2` **by use** — the thing's Wieldable claim names two hand slots
  because its *use* needs two (a bow: one hand holds, one draws; a
  musket; a spear couched as a pike). Authored, and only for use: a
  weapon row whose two hands were only about weight lost its claim — the
  holder decides;
- else `1`.

`wieldPlanFor(item, claim)` is the one question (the `*PlanFor` shape):
the slots it takes — a one-hand claim widened to two when the weight or
the reach says so — or `too-heavy` / `hands-on-cart` / `hands-needed`.
`equip` asks it, and words the body's own refusal.

## The tiers — hand · ready · packed

`tierOf(thing): 'hand' | 'ready' | 'packed' | 'elsewhere'`, read off the
slots and the placement pairs, never stored:

| tier | what | a … |
|---|---|---|
| **hand** | in a hand slot | the mug you are holding |
| **ready** | worn; or hung `from` a worn host; or in a pocket of a worn garment (the garment's contents); or inside a container that is worn off the back, hung or pocketed | a key in a pocket, a holster on the belt, the canteen clipped to a pack's strap, the satchel at your side and what is in it |
| **packed** | inside a container in the `back` slot | the rucksack's contents — *take it off first* |
| **elsewhere** | not on this body | — |

The slot named `back` is what makes a pack's inside packed: a satchel
row claims `[torso]` and is ready, a rucksack claims `[back]`. Two rows,
no flag. `query --ledger` (`i`) buckets what is directly on a body by `tierOf`
(held · regions · worn · ready containers · packed containers — the
controller's `tiersOf`; a worn garment's pockets print by region under
*Ready*, and a **holder** is a container with an inside of its own — a
satchel, never a coat); the carrier's private `carryPlaces()` is the
arrival walk's order: a holder shaped for the thing's kind, worn pockets, then containers hung from or pocketed
on a worn host (a purse in a coat pocket is one — W9), then worn
containers with an inside off the back, then the back. A `CarryPlace` is
`{ host: Container; region? }` — a pocket's host is a Container now.

## ⭐⭐ The command def predicts everything (W10, owner 2026-11-14)

*"The command def is supposed to be the one document that, if you get your
head around it, you can predict everything a command will do."* Every
carrying behavior is either on the verb's view (an option, an arg, a help
sentence stating the ONE rule) or it is the player's own act. Retired by
this ruling: **homes** (a thing remembering where on you it lives), the
`readies:` arg field (the dispatcher bringing an instrument to hand), and
every implicit *put the held thing away to free a hand*. The player's
one-line conveniences are their own aliases and `def`s — see the
scripting-hand-economy slate.

### The tiers are hand · ready · packed — closed is a STATE, never a tier

*Ready* = you can get at it without taking anything off — a belt, a
pocket, a hung pouch, a worn satchel; *packed* = inside something you would
have to take off. ⭐ **A shut holder is shut wherever it is** — a pouch on
your belt, a box in your hand, a chest in the room: using or taking from it
is refused naming the next command (*"Your belt pouch is shut — `open
pouch` first."*, `canReach`), never an implicit open. `open` resolves at
once; in a fight it is the unfasten beat you pay.

### Closures

A closure is per object, the author's (lens 2): a `Sealable` holder (a
pouch with a flap, a satchel, a pack, a chest) opens and shuts by
`open`/`close` and may refuse to reclose (a wax seal — `Sealable.canClose`);
an open hook or tool loop (`from`) has no closure; and **a chamber whose
entry says `closes: true`** (a buttoned pocket) is a closure the player
drives — `Placing.isRegionOpen(name)` / `fastenRegion(name, shut)` (the object's act, refused `does-not-close` / `already-…`), backed
by the host's own persistent `shutRegions` field (per instance; `null` =
the row's default, *every `closes` chamber starts shut*). `open "breast
pocket"` / `close "breast pocket"` — the binder's `via.region` names the
chamber. Its readers: `Containable.reachRefusal` (`enclosed-shut { by,
region }`), `PerceptionLogic.holdersAdmit` (a shut chamber hides), pressing
(`height: !isRegionOpen`), and `look`/`query --ledger` (*(shut)*).

## ⭐⭐ Putting away — `stow`

`stow(held): ArrivalRefusal | null` is **the route off the hands**: the
first of the carrier's places that takes it, in D11's order and nothing
else — a holder shaped for its kind (a scabbard, a holster: it takes nothing
else, so it reorders only for its own thing — finalize run 15), a pocket of a worn garment, a hook on a worn host (`from`), a
container hung or pocketed there, a worn bag off the back, the pack on the
back (`carryPlaces()`). Nothing takes it → `nowhere-to-carry { item, held,
tried }`, **nothing moves, it stays in the hand** — never the floor (that
is `drop`), never a guess. `unwield` is this act said in its help, and
tells you where the thing went (*You put up the sword — on the belt.*);
nothing taking it → *There is nowhere on you to put the mug — it stays in
your hand.* A thing that came out of a pocket does NOT go back to "its"
pocket: there are no homes.

## ⭐⭐ Readying — into a free hand, or a refusal

`ready(item)` brings a thing into a **free** hand (D-W10-3):

- already in hand → ok;
- **ready** tier with the hand(s) it needs free → `bringToHand` seats it
  and returns a `ReadiedOutcome` (`{ ok, item, moved, from }`);
- hands in the way → **`hands-needed { hands, held, away? }`**, worded
  *"That takes a free hand — you're holding the mug and the lantern. `drop
  lantern` (free in a fight), or put one away first (`unwield lantern`,
  `put mug in "left pocket"`)."* — nothing is put away for you;
- **packed** → `packed` (*take it off first*);
- in the room: a fixture is used where it stands (ok); a portable thing →
  `not-on-you` (`get` is the act); in another body → `not-on-you`.

Its callers are exactly the explicit *keep it in hand* acts: `get X from
<something on you>` (the thing comes to a free hand — *You pull out your
compass from your pouch.*), `fight switch`, `fight draw`, and combat's
`performSwap`. ⭐ **No instrument arg reaches it**: an instrument is used
where it sits — in hand, in an OPEN holder on the ready tier, or within
reach — and stays there; the tool args carry `canReach`, whose refusals
name the next command (shut → *`open` it first*; packed → *take it off
first (`remove <pack>`)*). A defaulted tool list (`reachable:[mixin.ToolMixin]`)
is passed whole by `canReach` (`MqlManyResult.defaulted`) and narrowed by
the controller (`PerceptionApi.reachableAmong`).

### ⭐ A hand act in a fight is a guarded beat

*A verb never waits on a holder; combat's tick is the one budgeted clock*
(D-W10-13): outside a live session every hand act resolves at once. Inside
one, `put`, `get` (from a holder, or off the floor), `wield`, `unwield`,
`remove` of a pack, and `open`/`close` of a holder on you are the switch's
guarded beat (`CommandController.guarded` → `Combatant.beginHandAct`), and
`fight switch`/`fight draw` are `ready`'s switch — each priced by its holder
(`Combatant.handActSeconds` → `CombatLogic.handActSecondsFor`):

| holder | seconds (dial) |
|---|---|
| dropped | 0 — never a beat |
| a shaped holder (`shapedTo` — a holster, a scabbard), a `from` clip, a non-hand slot (the `sidearm` sheath) | `combat.draw.seconds` (1.5) |
| an open pocket, an unshaped pouch, the floor, a put-away (`unwield`) | `combat.reach.seconds` (3) |
| opening or shutting a holder or a buttoned chamber on you | `combat.unfasten.seconds` (3) |
| a worn pack off (`remove`) | `combat.switch.seconds` |
| a shut holder | refused — `open` it first (its own beat) |
| packed | refused |

The act lands at the beat's end (`WeaponSwitch`'s `land`); a second hand act
while one is live is `already-switching`. `fight draw`/`fight switch` reach
only what is **on your person** (`switch.target.scope: [inventory]`): a gun
in an open pack at your feet is `get gun from pack`, a beat of its own. See
[combat.md](./combat.md).

## Pockets — named placement regions

A pocket is a **region its garment offers**, never an object. `Garment`
composes `PlacingMixin` over `ContainerMixin` with a class default of no
placements and **no inside of its own** (`hasInterior() === false`), and a
row names its pockets as entries over the `/platform/placements/in`
document (`encloses: true`, words *in* / *into*):

```yaml
placements:
  - { member: /platform/placements/in, name: left pocket,
      mouth: { width: 0.12, height: 0.035 }, volumeM3: 0.0006 }
```

⭐⭐ **A pocket is a region of its garment, which is a Container; the key
is the coat's contents** (owner, 2026-11-13 — *an enclosing relation is
containment; a supporting or hanging relation is a pair*; size and
carrying W9). The pocketed thing has `container = the trousers` and the
pair `(trousers, left pocket)`; it walks key → trousers → body for every
tier, honesty and reach read. A shirt with no pockets holds nothing:
`put key in shirt` offers no region, and `move(key, shirt)` throws. The
entry's `mouth` and `volumeM3` are the tape measure
(realistic: a key, a coin, a matchbox go in a trouser pocket; a mug does
not). **`covered: true`** (an inner pocket) hides it from other people;
an outer pocket is covered when its garment is not outermost. **`closes:
true`** (a buttoned breast pocket) is a closure: it starts shut, the player
opens and shuts it (`open "breast pocket"`), and its state persists per
instance (§ Closures).

⭐ **Chambers that differ ask; chambers alike are the container's to fill**
(D-W10-6, owner final A). A chamber's *character* is what the world acts on
— its member (own air), `covered`, `closes` — never its size
(`Placing.chamberCharacter` / `alikeChambers`). `put key in trousers` (two
plain pockets) puts it in the first that takes it and names it; `put key in
coat` (two plain, one buttoned, one inside) asks *"Which — the left pocket,
the right pocket, the breast pocket (buttoned) or the inner pocket (inside)
of the coat?"*; `put key in "inner pocket"` names it. A bare `pocket`
matching several chambers of one garment binds the garment (no region) and
the same rule decides; the same word answered by two garments is the
binder's *which one?*. ⭐ A **programmatic** `move(key, coat)` with no
chamber named has nobody to ask: into alike chambers it takes the first
that fits; into chambers that DIFFER it **throws** a `ContainmentError`
naming the chambers (owner, finalize 2026-11-16) — the caller names one
(`move(key, coat, 'inner pocket')`). `look coat` shows each pocket once with its
difference word. ⭐ **A container is one interior OR a set of chambers,
never both** (D-W10-14): `Container.hasInterior()` is derived — true iff
it offers no enclosing region — and volume and fit are per chamber, never
summed. See [spatial.md](./spatial.md) § Chambers.

## The belt — every from-host

A **belt** is a `Garment` row claiming `[waist]` with `placements:
[from]`. The same row shape is the bandolier and the chest harness
(`[torso]`), a pack's straps (`Pack` composes `Placing` — a canteen
clipped to a strap is ready while the inside is packed), the pack saddle
(`back:1` and `from` — two panniers hang), a key ring (`round`, its
circumference), a pegboard and a tool rail. **No belt class.**

What hangs is bounded by **length** (each thing its hanging width, the
first always taken) and **strength** (the material's
`tensileStrengthMPa`); a worn belt's length is its wearer's waist
([size.md](./size.md)). `roomFor(region)` answers in words — *plenty*,
*one more pouch*, *a small thing*, *none* — and `i` prints it beside what
hangs there (*Hanging from your leather belt: … Room left: none.*).
`put holster on belt` → *"You clip the holster to your belt"*; `get
holster from belt` takes it off. A hung thing is **fastened**: it stays
put in water and zero g. Hung rigid metal **clanks** (below). What hangs
widens the body: hung things ride flat at the flanks by their thinnest
side, and a laden mule is refused at a narrow gate.

**Holders.** A holster, sheath, scabbard or quiver is a `Pack` row with
`shapedTo:` (a row path or a mixin name — the holster names the pistol
row, the quiver the arrow row, the canteen clip `BulkableMixin`), an
authored `mouth` and `shape: rod`. Draw-fast **derives** from `shapedTo`:
a holster on the belt draws faster than a pouch on the belt. A belt
pouch, purse, saddlebag or pannier is a `Pack` row that claims no body
slot and hangs; a bumbag claims `[waist]` (its own strap).

## ⭐ What is placed on a thing travels with it (D33)

`ContainmentLogic.moveCore` carries every thing whose pair names a moved
host into the host's new environment **with its pair kept** — a belt
taken off brings its pouches; a carried tray its cups. The followers
arrive *placed*, so a body receiving a belt admits them; the whole
bring-along is asked of the destination first, so a refusal moves nothing
(a body's `arrivalFor` admits a placed arrival). ⭐ A coat's pocketed key
is NOT a follower (W9): it is the coat's contents, and goes where the
coat goes as contents do — taken off, given away or hung in a wardrobe, it
is still in its pocket. See [spatial.md](./spatial.md).

## Leading and hitching

- **Hitching takes hands.** `hitch` refuses `hands-on-cart` when
  `freeHands()` is fewer than the cart's `handedness`; while hitched,
  `freeHands()` is reduced by it and an arrival or `ready` says so (*"Your
  hands are on the handcart — `unhitch` it first."*).
  `Drivable.getDriveHands()` is a hauled host's `handedness`, else its
  authorable `driveHands` (default 2 — reins). See
  [conveyance.md](./conveyance.md).
- **Leading.** `tie rope to mule` resolves to the **halter** — the first
  `Tieable` in `Attired.wornStack()` (the `/trade/tanning/thing/Halter`
  class; `Carrying.halter()` retired, W8): the rope's end hangs `from` the
  halter and the free end comes to the leader's hand. A tieable with no
  `from` has the rope moved onto the animal, which places it; none →
  `no-halter` (*"the mule wears nothing to tie to"*). `Rope.tieTo` leads a
  body that is handless **or wears a halter** — any body, carrier or not
  (a haltered hog is led)  — so a dog (whose mouth is a hand) is led on its
  collar, and a person handed a rope takes it in their hands.
  `Mobile.crossExit` leads: for each held line end whose far end is on a
  led body in the same place, it pre-checks `exit.canTraverse(animal,
  its default mode)` — a refusal refuses the leader (*"The mule won't
  follow you: …"*) — then the animal follows. The rope's length is not
  modeled across rooms.

## What the body stops with

`releaseHeld(slots?)` moves each hand occupant to the enclosing place
with a scene line. `Vitals` calls it when the body **goes under** (a fall
from conscious read in `reconcileConditions` — dying and sedation read
unconscious) and on **severance** of a hand's part. Worn things stay.
Death still moves everything to the corpse. Logout drops nothing.

## Restores and guests

- **No first-load placement.** D32's `placeLoose` was cut at finalize
  (above): nothing loose is ever captured, so nothing needs placing.
- **An owned host keeps its pairs across a restart**: a pair whose host
  is an owner-persisted good is captured as `hostChattelId`, and an owned
  good placed on a host records `placedOn` on its estate entry; the
  restore re-places both after re-wearing (a worn belt's length is its
  wearer's waist, so pouches re-hang as worn).
- **A guest's borrowed things go home** (D37). Before a guest is reaped,
  `returnBorrowed()`: a good titled to someone else goes to its owner's
  hands (else set down where they stand), anything untitled the guest
  held or pocketed is dropped where it stands, what it wears is reaped
  with it.

## Noise, awkwardness, running

- **`gearNoise(): 0 | 1`** — 1 when a held thing, or one hung `from` a
  worn host, is rigid and rings (`ringSound().rings`, else a `metal`
  tag): a sword on a belt, a lantern in hand, a key ring. A thing in a
  pocket or a shut pouch is silent. `LocomotionLogic.emissionAt` lifts the
  mode's `noiseLevel` one band — what `sneak` and stealth read.
- **`awkwardnessOf(item)`** — the burden gauge's (`LoadBearing`, W8):
  `1 + AWKWARD_LENGTH × max(0, longest/stature − 1) + AWKWARD_BULK ×
  volume/bodyVolume`, multiplying a held load.
- **`run` refuses `awkward-load`** naming the thing when a held thing is
  longer than `RUN_AWKWARD` (1.0) × stature or more awkward than
  `RUN_AWKWARD_FACTOR` (1.5): you walk the pole. See
  [locomotion.md](./locomotion.md).

## Seeing what others carry

Perception decides, not MQL ([perception.md](./perception.md)): on
another body you see what is in their hands, the outermost layer at each
slot, what hangs from their belt, and an uncovered, unfastened pocket's
contents — never an inner pocket, a buttoned one, or a shut bag.
`query --ledger <person>:i` lists exactly that. `search <body>` on a body that
cannot resist (a corpse, the unconscious) finds what its pockets cover;
a conscious one refuses (*"Bob won't let you"*).

## The verbs

| verb | what it does here |
|---|---|
| `get` | `move` — a hand, then a pocket or bag it fits; a no-slot pouch onto the belt. When the routing would only set it down, `get` refuses first (`wouldSetDown`) and nothing moves — off a wall or out of a slot included. `get X from <on you>` → `bringToHand` into a free hand, or `hands-needed` |
| `put X in/on/from Y` | onto a host or into a container; chambers by the rule above; `put <tack> on <beast>` seats tack on a body that cannot dress itself (`Slotted.openSlotFor`) |
| `give X to Y` | the release gate, then `move(X, Y)` — onto them, a bag they carry, or set down at their feet with the line (a mule handed a saddle sets it down — tack is `put saddle on mule`); a non-carrier takes nothing, refused before the release |
| `unwield` | `stow` — off the hands by the one order, told where; nothing takes it → refused, it stays in hand |
| `drop` | to the place; instant, and free in a fight |
| `query --ledger me:i` (the `i` and `inventory` default aliases — W10h) | by tier — *In hand* · *Ready* (each pocket and hanging member by name with the room left along a hanging length, each ready container with its perceived contents) · *Worn* · *Packed* (each pack as *N things, M kg* unless the set names what is in it — `me:I`, `rucksack:i`) — then the burden in words. Display options `--weight` `--flat`; narrowing is the query's (`me:i:held`, `me:i:worn`, `me:i:carried`; no `ready`/`packed` word); `query --ledger <container>:i`, `query --ledger <person>:i` |
| `wear` / `wield` / `equip` | `seatInSlots` by the claim; the grip by `wieldPlanFor` |
| `fight draw` / `fight switch` | `ready` in a session |
| an instrument verb (`measure`, `analyze`, `read`, `light`, `pour`, `fill`, `shape`, `cut`, `fit` …) | uses the tool where it sits; `canReach` refuses shut (*`open` it first*) and packed (*take it off first*) |
| `stoop` / `crawl` | lower profiles through low ways ([size.md](./size.md)) |

The player's guide is the `carrying` wiki guide (base-library
`guide/carrying`).

## ⭐ The census gates

- ~~**`lint:body-moves`**~~ — **retired W8**: it counted the moves onto a
  body so each could be ported to `receive`; with `move` the receiving act
  there is nothing to port, and a body with no honest place throws at run
  time, naming why.
- **`lint:perceived-surfaces`** counts every surface that shows a viewer
  a list it never asked `PerceptionApi.perceived` about. 32 → **0** at W3.

See lint-family.md.

## Dials — `CARRYING_DEFAULTS`

`ONE_HAND_FRACTION` 0.3 (of carry capacity — a 70 kg human holds 10.5 kg
in one hand) · `TWO_HAND_LENGTH` 0.6 (of stature — a 1.75 m human's
1.05 m). The awkwardness dials (`AWKWARD_LENGTH` 0.5 · `AWKWARD_BULK` 1.0 ·
`RUN_AWKWARD` 1.0 · `RUN_AWKWARD_FACTOR` 1.5) are `LOAD_BEARING_DEFAULTS`'. A body whose carry capacity reads 0 (no mass
yet) is not judged by weight. The combat seconds are app settings
(`combat.draw.seconds`, `combat.reach.seconds`, `combat.unfasten.seconds`,
`combat.switch.seconds`).

## Deferred

- **The ledger's goods-first for cash** — coins minted after the post go
  through `move` (routed: to you, your purse or bag, else at your feet);
  ordering the goods first would mint before the ledger and destruct on a
  refused post — a banking restructure.
- **A competence term in readying** (the combat slate).
- **Autocomplete** must obey the perception filter when it exists.
- **The roof rack** (an outside surface on an `Exitable` host).
- **A hand into a pack someone WEARS** — the pickpocket's act — is
  refused today (reach never goes into another body, D17); it is designed
  in policing-slate § Rails.
- **The scripting conveniences** the engine no longer performs (try-else,
  waiting on a beat, a fits predicate, aliases with arguments and a
  `$1`-else-default form) →
  scripting-hand-economy-slate.

## History

- **Size and carrying (2026-11-11 → 2026-11-12, `39d4ba23e..50f562ed4`).**
  The subsystem landed in eight waves: the census and `receive` (W0),
  hands, tiers, homes, readying and the kernel ports (W4), pockets, worn
  and hung containers, the tiered inventory and the pack ports (W5), the
  gear rows and the starter kit (W6), and **the flip** (W7) — the
  `loose-on-body` veto, `receive` as one resolution, the three idioms
  across every site, `held` = a hand everywhere, first-load placement and
  the guest return. Before it, `get` claimed no hand: a thing sat loose
  in the body's contents and paid a held surcharge
  ([encumbrance.md](./encumbrance.md)), and *hand slots were for
  activities, not storage* ([embodiment.md](./embodiment.md)).
- **W8 — `move` just works (owner review, 2026-11-12).** The owner read
  `Carrying` (35 members) as *"a camel or hydra or blind man's elephant"*
  and set the shape: *"if I'm a content author and I need to just move
  something into someone's inventory … I can call `move(item, whoever)`
  and I don't really have to worry about anything else"* — and *"the one
  thing I don't want is for move to lie."* So: the body places what
  arrives through `Container.arrivalFor`, or `move` throws; `canReceive` is
  on every container; the `receive` family is deleted (no synonym for
  `move`); the invariant is every `Creature`'s and fails closed; the
  capability composes on the four carrier classes; the hands are
  `Attired`'s, awkwardness the gauge's, the switch's seconds combat's, and
  the words the controller's. `Carrying`'s interface is 14 members.
- **W8i — a carrier routes (owner, 2026-11-13).** W8's `move(item, body)`
  silently turned into `move(item, bag)` — the move to the body never
  happened. Four rulings in one day settled it: `move` is the one call
  and *a carrier routes what arrives* — it reaches the body (a true move,
  its witnesses fire) and the body moves it on into a bag, else sets it
  down with the line; *"this shouldn't be on the caller — it should be on
  move()."* `canReceive` and the 68 hand-written made fallbacks are gone;
  the bypass is `carrier.directMove(item, route?)` in `Arrival`'s
  vocabulary, and a handed act moves the goods FIRST and the money after.
  `Carrying`'s interface is 16 (`routeArrival`, gated to the chokepoint,
  and `directMove`).
- **W10i — `move` is the only call (owner, 2026-11-15).** `directMove`
  is retired with its `CarryRoute` and the `how.onBody` read. Every site
  that moved a thing onto a body — `give`, `buy`, a tip jar, a store
  withdrawal, a reclaim, an untie, `put X in <person>`, `equip` out of a
  wardrobe, the daypack, a lead onto a carrier — calls
  `ContainmentApi.move(item, body)`, and what the body cannot carry is set
  down at its feet with the routing's line. ⚠ D-W7a's *"a handed thing is
  pre-checked before money moves"* is **retired**: the shelf settles, then
  hands over (`handBack` is gone with the refusal path). `get` keeps its
  refusal — through `wouldSetDown`, the routing's own answer read dry —
  because a pick-up that is only set back down changes nothing.
  `lint:carry-moves` holds the name and the `isCarrying(x) ? … :
  move(…, x)` branch at 0.
- **Finalize (2026-11-16; the whole build is `39d4ba23e..HEAD`, MR
  !576).** Owner rulings: the pre-flip restore is **cut** (`placeLoose`,
  the residence overflow, `placesLooseOnRestore` — no migrations), so
  `Carrying`'s interface is **12** plus the `arrivalFor` override;
  `wornBulk` moves to `Attired`; `seatInSlots` stays the explicit *occupy
  these slots* act; the routing's set-down line stays in the mixin; and a
  programmatic `move` into a host whose chambers DIFFER, with no chamber
  named, **throws** ([spatial.md](./spatial.md)). The review waves W8–W10k
  (Carrying is one job · a carrier routes · pockets are chambers · the
  cuts · `ShapedMixin` · `inventory` retired for `query --ledger` ·
  `directMove` retired · `shape` documents · `lint:data-ideas` /
  `lint:view-keys`) are recorded above and in the build's commits.
