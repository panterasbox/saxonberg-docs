# Taps — a renewable yield off a living thing

⭐⭐ **A tap is a reservoir with a recharge law, drawn by an act, that
credits a Discipline.** That is the RGO law, and five shipped things
satisfy it: a cow's milk, a hen's eggs, a ewe's fleece, a colony's
honey, and a birch's sap. This doc owns the mechanism they share and
the places they genuinely differ.

Shipped by the taps build (2026-10-01). Before it, the mechanism was
`trade-ranching`'s and the ACT did not exist — every take was instant,
vesselless, and pointed at an animal by a verb that already knew what it
would get.

---

## ⭐⭐⭐ The feedback law — what actually distinguishes these products

`TapSpec.behaviour` says what NEGLECT costs. It does not say the thing
that decides how each product should PLAY, which is **whether the act of
taking feeds back on the rate**:

| | feedback | the judgment, and where it lives |
|---|---|---|
| **milk** | ⭐ direct — lactation is demand-driven, so removal stimulates synthesis and residual suppresses it | ⚠ and therefore **nothing to decide at the act.** A take always empties her. What the player trades is **attendance against her rate** — labour — and ⛔ **nothing in the game discharges it yet**: the standing-instruction relief was cut in review (§ The relief), so AC 7 is unmet. `look` bands the window clock so the loss is at least visible coming |
| **eggs** | ⭐ through a state the act PREVENTS — a hen is an indeterminate layer, so a clutch left standing makes her brood and stop | `TapState.brooding` + `fullSince`, armed by `broodAfterDays`. Take the clutch and she starts again |
| **wool · honey · sap** | none | nothing to decide at the act; `TapState.worst` records what the YEAR put in, read at the take |

⚠⚠ **An earlier draft of this design asserted the opposite** — that all
of them were one mechanism with different dials (*"make it work like
honey"*). They are not, and the biology is what says so. Every per-tap
field exists because of that table, and each is read by exactly ONE
behaviour.

⚠ Two mechanisms were specified and then DELETED for failing it, which
is worth recording because both came from the plan rather than the
requirements:

- **`TapState.vigour`** (a milk suppression curve). Unimplementable:
  `ceiling = perGameDay × windowDays`, so a cow fills to her ceiling
  exactly as her window closes, and the region where she sits full and
  suppresses herself is *empty by construction* — it begins where the
  neglect cliff already fires.
- **`shear --quick`** (speed for quality). The table's third row says
  wool has nothing to decide at the act, so it was a judgment the
  biology does not put there.

---

## The vocabulary

`TapSpec` on `Species.production[]` (kernel, authorable). A tap is a
species fact, which is already the engine's belief about animals.

| field | what it decides |
|---|---|
| `key` | what comes out — the word the verbs and the register speak |
| `yieldRow` | the row a take mints — ⚠ **or the MATERIAL, when `yieldShape` is `volume`**: the vessel is the object, so there is nothing to clone |
| `perGameDay` | units per GAME day at full production. ⚠ Never "daily" — a game day is two real hours at the shipped 12× scale |
| `behaviour` | `accrue` · `expire` · `continuous` — what neglect costs |
| `windowDays` | the `expire` cliff, and the `accrue` ceiling's divisor |
| `window?` | ⭐ **when the tap is open at all.** Absent = `always` |
| `yieldShape?` | `mass` (default) · `volume` · `count` — ⭐ and therefore **whether a vessel is needed**, never a second flag |
| `capUnits?` | a `continuous` ceiling; past it the growth is lost and `look` says so |
| `broodAfterDays?` | the hen's clutch clock. Absent = never broods |
| `takeMs?` | how long the take holds the hands. Absent = by shape |

### `TapWindowSpec` — a season is DATA

```
always | event | photoperiod | biome | weather
```

`ProducingMixin.tapWindow(key)` evaluates it, so **a new season is a row
and not a code path.**

- **`event`** — opened by `freshen()`, closed by drying off. Milk's
  shape: a lactation is not a season, and a cow is near-aseasonal.
- **`photoperiod`** — a daylength band, read through the SYNC twin
  `CelestialApi.daylightSecondsFor(EARTH_LIKE, CAMPUS_LATITUDE, now)`
  (a reconcile cannot await, and the async face only awaits a zone field
  guarded to `EARTH_LIKE` anyway). ⭐ A wrapping band is legal.
- **`biome`** — the HOST answers (`biomeWindowOpen`). A hive reads its
  own forage census, which is the first RGO whose reservoir is somebody
  else's land.
- **`weather`** — daylength AND the host's own temperature. ⭐⭐⭐
  `rising` discriminates spring from autumn, **which cross the same
  daylength band and are not the same season for a tree**: without it a
  birch would run twice a year, and nothing in the game would ever have
  reported it.

⚠ **The tri-state rule:** a term this world does not model reads as
OPEN, never closed. A host that answers no temperature is gated by
daylength alone rather than silently dead.

⚠ The window gates the **fill** and nothing else. What is already
standing stays takeable — a season ending is not a reason to confiscate
what the season produced. And it is sampled **at the read** (the `Stand`
growth-factor form), so a jump across a boundary bills the whole
interval at the boundary's own answer. Honest-cheap, and stated.

### `TapClosedReason` — ⭐ none of them is a failure

`before-season · after-season · cold · warm · dried-off · brooding ·
no-forage`. The first four are the calendar and the weather,
`dried-off` is a consequence already spent, and `brooding` is a bird's
own decision. So `TapActController` renders a `season-*` reason through
`EngagedActController.inform()` — the same channel, a different
register, **and no `controller-rejected` note.** Rendering the calendar
in the rejected voice would make it read as the player's mistake.

⚠ **No kernel refusal contains a digit**, and the tree's never say the
noun *tap* (it is bound on nine bar fixtures, so a sugaring refusal
saying *the tap* could read as being about a beer engine). It says
*spile* and *stem*.

---

## The act

`lib/husbandry/Tappable.ts` — a **sibling** of `Workable`, sharing its
result types (`WorkPlan`/`WorkRefusal`/`WorkPrognosis`/`WorkResult`) and
not extending it. ⚠ `planWork(by, tool, what)` has no vessel slot, and
more importantly *claiming a lactating animal is worked ground* is a
host-placement lie: ground prices its own pace because ground is what is
being changed, and a cow is not improved by being milked.

`ProducingMixin` implements both halves by default, so a cow, a hive and
a tree get the whole act for free and override only what is theirs.

**⭐ The credit is the HOST's** (`WorkResult.credit`, via the
`tapCredit(key)` hook): the animal answers `stockmanship` with a
difficulty read off its own handling, the hive `apiculture`, the tree
`silviculture`. That is what lets a kernel controller earn a trade's
competence **without knowing the trade exists** — ground's own seam,
reused — and it is what retired the old `TapController.discipline()`.

### The controllers

```
lib/command/EngagedActController        (abstract — the hands)
  ├── platform/idea/cmd/ground/GroundWorkController
  │     └── WorkedActController            (dig · split)
  └── platform/idea/cmd/inventory/TapActController
        ├── MilkController · ShearController · GatherController  (ranching)
        ├── RobController                                        (apiculture)
        └── TapController                                        (forestry)
```

`EngagedActController` holds the endurance check, the `hands`
engagement, the three start-failure reasons, `decline` and `inform`. It
was promoted at its **third** consumer, which is the stated trigger.

`TapActController` lives in `inventory/` beside `HarvestController` —
taking a renewable yield off a living thing is the harvest family. A
subclass names ONE string (`tapKey()`) and its *nothing here* sentence.
⚠ **The god-verb guard:** if a new tap needs that file to branch on what
kind of tap it is, the design is wrong.

### ⚠⚠ Two things the act must get right, both learned the hard way

1. **The completion is a MODULE function, never `this.<method>`.** A
   controller is one ephemeral clone per execution and the dispatcher
   destructs it in a `finally` the moment `execute` returns — while the
   engagement is still pending. A completion calling back into it runs
   on a destroyed Stuff and the proxy answers with a **silent no-op**:
   the scene plays, nothing is minted, nobody can tell. The dig/split
   and smelt builds each shipped that bug and a live drive found it.
2. **The take re-checks co-location where it LANDS.** Nothing in the
   scheduler cancels a `hands` engagement on movement
   (`LocomotionLogic.engageAround` touches only the engaged *mode*), so
   walk away from a cow and the act must produce *nothing came of it*
   with no state change. ⚠ The check walks ONE level out on the
   subject's side, because a tappable tree stands in a panel standing in
   the room — without that, every sugarbush take lands as *you are not
   there any more*.

`completeTap` also **re-reads the tap** rather than trusting the plan's
numbers: the interval between planning and landing is real game time and
a season can close inside it.

---

## Host placement

| thing | host | the claim |
|---|---|---|
| `ProducingMixin` | `lib/husbandry/` — composed by `Livestock`, `Hive`, `SapStandard` only | ⛔ Not `Plant` (every houseplant would give something), not `Creature`. Promoted because its composers have **no common pack ancestor** |
| `TapState.worst` · `brooding` · `fullSince` | per-TAP on `tapState` | a species may carry two continuous taps; *the clutch* is what broods her, not a hen flag |
| the `Tappable` defaults | on `ProducingMixin` | every producer is tappable with no per-host code; overrides are the bespoke case |
| `spiles` | `SapStandard` | per-instance: a tree carries its wounds across a bounce and a felled one takes them to the ground |
| `tap.yaml` affordance | `SapStandard.peers` **and** `StandMixin.inventory` | ⛔ Not `Panel.peers` — a hazel panel would offer it and the controller would have to un-promise it, which is the host-placement tell |

⚠ **No migration, ever.** A `TapState` persisted before `worst`,
`brooding` and `fullSince` existed comes back missing them; they default
on read and the first reconcile writes them back.

---

## ⭐⭐ The three-way contest — the design is legible from the refusals

| where | `tap` | what it teaches |
|---|---|---|
| beside a `SapStandard` in a panel | **works** | you tap a stem |
| in a `Wood` | **afforded, and refuses**: *"A stand is a number of trees, not a stem. A spile goes into one tree you can put your hand on."* | ⭐ the wood affords a verb it cannot satisfy *in order to say that* — a sentence only something that answers the verb can say |
| at the treeline (no `Wood`) | not afforded — the platform's not-here answer, **identical for `fell`** | the parity is deliberate: the verbs are legible from where they ARE |

⚠ The tree reaches a player in the room because the affordance walk goes
one level into an **open container standing there**, and a `Panel` is a
`GardenBed` with no `Sealable`. **If a future panel is ever made
sealable, every sugarbush affordance dies silently.**

---

## The clock seam

`WorldClockApi.advance(by)` moves world-time forward and **drains** the
skipped interval: one-shots fire once, an `every` once per missed
period, in deadline order. Reachable from the `eval` sandbox, which is
the code-trust axis — `setScale` was always reachable from there and
this is the same authority stated honestly. ⛔ No new verb, no
`isWizard` check; `shutdown` stays `SystemRoot`, so an eval still cannot
freeze the world.

⚠ It **throws while the clock is paused**: a paused clock fires nothing,
so a jump there would bank the game-time and strand every schedule in
the interval — the exact silent skip the drain exists to prevent.

⚠ A cascade does **not** catch up: a callback that re-arms off `now`
lands after the jumped time, because that is where `now` is. An `every`
catches up; a chained `after` does not.

⭐ Why it matters here: the taps are reconcile-on-read and need nothing
from the clock, but a *scheduled* act does — and without this seam no
drive could ever walk a season. `taps.dirty.wire.test.ts` is the first
drive in the repo that does.

### ⛔⛔⛔ No QUARANTINED context may mutate world time

⚠⚠ The first version of this shipped `advance` **ungated**, justified by
a sentence claiming *"`setScale` was always reachable from the sandbox"*
— which was **false**: before that binding nothing in the sandbox could
touch the clock at all, and the only reference to `setScale` outside the
clock's own files was that sentence asserting it. Worse, the binding was
the whole `WorldClockApi`, so eval'd code also got `pause`, `resume`,
`setScale` and `restore`. Raised in review.

⭐ **The argument that settles it is `shutdown`'s own.** `shutdown` is
`SystemRoot`-gated *because nothing in-world may FREEZE world-time* — so
by identical reasoning nothing quarantined may **skip** it, slow it,
pause it or re-anchor it. `advance` is that comment's inverse and was
ungated right beside it.

So every clock **mutator** now calls
`WorldClockRegistry.assertNotQuarantined`:

| caller | may mutate the clock? |
|---|---|
| a **wire circle** (`runScoped` → `circleScope` planted) | ⛔ **no** — refused, naming why |
| a **governed** jurisdiction (`runGoverned` → `jurisdictionBound`) | ⭐ yes, and the eval path already receipts it (provenance + a `sandbox.eval.governed` line) |
| boot, a schedule, a test (no scope planted) | yes |
| any of them, for a **READ** (`getNow`, `getScale`) | ⭐ yes — containment is about effects escaping, not secrecy, and a circle that cannot read the clock cannot simulate anything |

⭐⭐ It is a **containment** property, not a permission one: the sandbox
promises a circle's effects stay inside the circle, and **there is no
per-circle clock** — world time is global and shared by every player, so
a clock mutation from inside a quarantine breaches containment by
construction, however well-intentioned the caller.

⚠ And `advance` writes a `WorldClockApi: ADVANCED by …` **server-log**
line, so a jump is never silent. The server log rather than `MudlogApi`
because mudlog needs a recipient and `advance` can be reached from boot
or a test where there is nobody to tell. A jump is irreversible (time
only runs forward) and ages every reconcile-on-read system in the realm
at once, which makes a silent one the hardest thing in the game to
diagnose after the fact.

---

## ⛔⛔ The relief — CUT IN REVIEW, and it is milk's unanswered question

**Milk's answer was going to be the relief, and the relief is not in
this build.** W4 shipped `instruct keep <line>` plus a `keeps` brain so
a player's body could keep a milking round while they were offline. It
was removed before the MR merged, on two objections:

1. ⛔ **Scope.** It is offline automation, decided inside a build about
   tapping trees for syrup. Whether players may automate labour at all
   is a lens-level question — pedagogy (what Discipline does an absent
   body exercise?), participation, values, economy — and it needs its
   own conversation rather than a wave.
2. ⛔ **It contradicts the absent-body doctrine**: *automate only the
   AUTONOMIC (what any body does unattended), never the STRATEGIC (what
   a mind chooses); reflex is real, strategy-by-proxy is faking; the
   absent body holds LESS than an NPC brain, not a decision agent.*
   Milking a named cow into a named pail fails that test, and W4 gave a
   player body an actual NPC brain (`BehavedMixin` on `Avatar`).

⚠ **So acceptance criterion 7 is UNMET, and the dairy cow's attendance
cost is unrelieved.** That is the honest state of the design: the
feedback law says milk has nothing to decide at the act, so what the
player trades is attendance against her rate — *labour* — and nothing
in the game currently discharges it. `look` bands the window clock so
the loss is at least visible coming.

⭐ One finding is worth carrying forward, because it is about the
mechanism and not the policy: W4 took a **greedy free-form command
line** and replayed it through `forceCommand`, which means the brain
could issue *any* verb — so it needed a per-verb allowlist, and that
allowlist became a new `standing?: boolean` field on `CommandView` (a
598-view schema). ⭐⭐ **The field was a filter invented to re-narrow an
input that was accepted too wide.** A design that takes a TARGET and
lets the engine construct the take from the tap needs no allowlist
anywhere: a sale is then unreachable by construction. ⚠ And the
declaration proved exactly as forgettable as a list — there are five
`TapActController` subclasses and W4 set the flag on four, silently
omitting `shear`.

The design question lives at
`docs/slates/builds/standing-instructions-slate.md`.


---

## Deferred

- **Chicks / incubation** — `TapState.brooding` is the attach point.
- **The dry-off decision** — `window: {kind: event}` + `freshen()` is
  the seam; the breeding cycle calls it.
- ⛔ **The relief** — built in W4 and **cut in review** (scope, and the
  absent-body doctrine). AC 7 unmet; see § The relief above. →
  `standing-instructions-slate`.
- **Resin / pitch** — a `production:` row on `pinus/sylvestris` with
  `yieldShape: mass` and a `weather` opener, on `SapStandard`. Nothing in
  the kernel changes. → `tapping-slate`.
- **Real winter** — maple's freeze–thaw opener is the climate slate's
  first consumer, and the test that `TapWindowSpec` was declared as data
  for the right reason.
- **The fourth copy** — `ManualBuildController.engageStep` onto
  `EngagedActController`.

---

## Cross-references

[ranching.md](./ranching.md) · [apiculture.md](./apiculture.md) ·
[forestry.md](./forestry.md) · [husbandry.md](./husbandry.md) ·
[activity.md](./activity.md) · [maturation.md](./maturation.md) ·
[time.md](./time.md) · [textiles.md](./textiles.md) ·
[crafting.md](./crafting.md) (the sweetener vocabulary)
