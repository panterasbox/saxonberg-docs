# Dairying — making milk keep

The dairy trade, as a reader finds it. Shipped by the dairy build
(2026-10): the capability pack `trade-dairying` at `/trade/dairying`, the
Heart's Delight dairy farm and co-op creamery, and Rejection's goats.

⭐ **Milk is what you have to do something with TODAY.** It turns in about
a game day warm and keeps several days cool (`milk.yaml`: `Ea` 60 kJ/mol,
`a_w` 0.99 — derived from the shipped growth dials and pinned by
`Freshness.hold.test.ts`). Every product of the trade is a way of making
it keep, and every one rides shipped substrate: the pack ships **no verb,
no controller, no Api**.

## The chain

| act | what happens | substrate |
|---|---|---|
| `milk cow into pail` | litres of the animal's own milk; the milker's identity stamped on it; unwashed hands put `e-coli` in it and on the pail | `ProducingMixin` (`taps.md`) |
| `pour pail into pan`, wait, `pour pan into pail` | the charge **stands** (`restDays`), then the skim draws off and **stops at the cream** — a share derived from the milk's fat | `SettingPan` = `FractionatingMixin` (`fractionation.md`), schedule `milk-setting` |
| `pour pan into churn`, `churn churn` | between ~283 and 291 K the cream **breaks** (churned cream); too cold it will not come; too warm it comes greasy (`fair`) | `Churn` answers the kernel's `HandWorked` shape; the `stir` view's `churn` verb |
| `pour churn into jug` | buttermilk draws off; **butter** is the residue — a `fat`, so every fried recipe takes it | schedule `churning` |
| `heat vat` | the vat is SET ON the lit fire and warms; the thermometer reads it | `heat`'s placing arm (`crafting.md`) |
| `pour crock into vat` | a crock of whey IS a living lactic culture: the pour pitches the batch; milk poured into the crock feeds it | profile `whey-starter` (`kind: culture`) |
| `pour rennet into vat` | the batch **rekeys** to renneted milk — a firm curd — whatever the order of rennet and starter | profile `soft-curd`'s `transforms` (`maturation.md`) |
| (wait) | the batch sets: the vat holds whey over a curd whose share derives from the milk's fat and protein | profiles `soft-curd` / `firm-curd`, `leesShare` |
| `pour vat into pail` | the whey comes off down to the curd; poured into the crock it is next time's starter | the lees split |
| `pour vat into hoop`, `get stone`, `put stone on hoop`, wait, `get wheel` | the curd **drains and sets** in the hoop's shape; pressed, it is drier and keeps longer | `Hoop` = a kernel `Mould` (`mould.md`); the curds' `drainSetting` |
| `cure wheel` | salt binds water — the hard wheel's second hurdle; the same wheel comes back, still what it was made from | trade-cooking's `cure`, which finds dairying's `salt-wheel` recipe by its item slot |
| `put wheel on rack` (the cheese room) | the wheel **ripens** by its own temperature: young · coming on · ripe · (soft) over · (too warm) dead | `Wheel` = `RipeningMixin(Provision)` (`maturation.md` § the discrete twin) |
| `split wheel`, `consign wedge` | wedges carrying the wheel's state, sold on the co-op shelf | `Splittable`; `retail.md` |

## Heat, and its cost

A vat on the fire integrates its own trajectory, so heat outside a recipe
**already kills**: `Freshness`, `Contamination` and the batch's culture
each die above their own threshold. ⭐ **The scald mark**: a batch whose
peak temperature since the fill passed its profile's `scaldAboveK`
(355 K for a curd) sets `poor` — pasteurizing at about 63 °C is safe;
boiling is not. ⚠ Milk scalded in a POT and poured into the vat arrives
unmarked (the mark is the batch's, not the payload's) — the deferred
`BulkPayload.peakK` seam.

## A live ferment holds spoilage

While a batch with a strain is converting — or a culture is alive, or a
wheel is ripening — **the population is the flora**, and the spoilage
gauge holds (`SpoilageHolding`, asked by `Freshness`). ⚠ Contamination is
never held: a raw-milk cheese keeps its pathogens, which is the point of
pasteurizing.

## The species reach the cheese through the numbers

Three milks share the tag `milk` (cow, ewe, goat), so one schedule and
one profile serve them all. Cream and curd shares are
`CompositionShare`s evaluated against each milk's own fat and protein —
ewe's milk gives about twice the cow's curd with nothing authored per
species. A batch remembers what it was made from (`madeFrom`) and the
wheel's taste names it (*made from goat's milk*). **A fourth dairy animal is a material row and a
tap line.**

## The herd

A tap has a **sex** (`TapSpec.sex` — a bull refuses in his own voice) and
a **rest** (`restDays` — a cow dried off and returned is still dry if
redrafted inside it, and back in milk after). A herd book's
`femaleFraction` is a herd fact; a drafted head's seeded sex is applied at
the draft. A **vell** comes only off a young head (`ButcheryYield.maxAgeDays`)
— the dairy farm's calf register is the creamery's rennet supply.

## The venues

- **Heart's Delight** — the dairy farm on the flats south of the gate (the
  byre: herd book, calf register, a pail, a block for the bull calves),
  titled to the dairy household; the **co-op creamery** southwest of the
  gate (pans, churn, vat, hoops, the starter crock, the rennet jar, a dairy
  fire, press stones, a salting trough, the co-op's shelf), titled to the
  `creamery-coop` organization; the **cheese room** below it at 284 K.
- **Rejection** — three nannies in a pen west of the yard gate, with a
  pail, a pan, a crock, a small vat and a hoop. Rows only: Rejection still
  ships no TypeScript.

⭐ **A second creamery is rows.** Nothing here names a venue.

## Discipline

`dairying` (ISCED 0721), a specialization of `fermenting`. Credited by the
setting and churning draws (the schedules' `discipline`), a fill into the
hoop (the mould's `discipline`), and the rennet, scald and ricotta
recipes. Competence grades; it never gates.

## Deferred

- Staffing the twice-a-day tap → `ranching-slate`.
- NPCs buying cheese and butter → `nutrition-and-fitness-slate`.
- Whey to pigs, feeding anything → `ranching-slate`.
- Calving as the lactation's cause (`freshen` is the seam) →
  `ranching-slate`.
- Fluid milk sold cold, the separator, HTST → `cold-chain-slate`.
- Salt in the hoop; rind care, bloomy and blue moulds; `BulkPayload.peakK`
  → `dairy-slate`.
- Cured meats composing `RipeningMixin` → `cooking-slate`.
- ⚠ The kill curve is about half as temperature-sensitive as real milk
  pathogens (flash pasteurization under-kills) → a finding on
  `spoilage.md`.

## History

- **2026-10-16 — the browser walks.** Three pails in the byre and two at
  the goat pen (more than one milker); the creamery and its cheese room
  each HANG a lantern (`adornments:` — a lamp loose among the props lights
  nothing), and the creamery authors `extent: 5`, because every Delight
  room otherwise takes the zone's 20 m cell and a 400 m² room reads
  `very-dim` under any lantern. The cheese room keeps the cell: sized, its
  racked wheel spoiled (suspected atmosphere, not traced).
