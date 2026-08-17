# Ranged — the band ladder, and delivery across it

The source of truth for ranged combat. **Wave 1 is shipped**: the
engagement band ladder, the arena that caps it, the two band-change
gambits, the delivery contract, the commit-time consent gate, and one
carrier — the thrown flask. Waves 2–4 (cover, armor, archery, guns) are
rostered at the end of this doc.

Seeded by ranged-slate.md. Scope was
agreed in a requirements doc of 44 decisions and 56 acceptance criteria,
retired at the pre-merge sweep — this doc is the live reference, and the
decisions that still bind are stated here in their own right.

## The one abstraction

> **A launcher imparts energy to a projectile that crosses a band gap
> and applies a payload on arrival.**

Bows, slings, thrown blades, guns and thrown flasks are all that shape.
Combat, placement and the wound model consume **one derived tuple** and
therefore cannot tell an arrow from a bullet from a flask.

## Bands are relationships, not positions

`RangeState` is `close · reach · near · far`, ascending
(`lib/combat/RangeBand.ts`). A band means **how engaged are these two
with each other**, not how many metres apart they stand.

| Band | What it is |
|---|---|
| `close` | the clinch; a dagger or unarmed owns it |
| `reach` | a polearm holds you off |
| `near` | thrown blade, sling, flask, sidearm |
| `far` | bow, crossbow, long gun |

**Bands do not compose.** A may be `far` from the archer while B is
`close` to the archer and A and B are `close` to each other. There is no
triangle inequality, no derived position, and nothing propagates. Read
it as prose and it is obviously right: *"you're keeping the swordsman at
bowshot while his partner is already on top of you."*

Ranges are **symmetric per pair** — `CombatGraph.setRange` writes both
directed edges — and per-pair asymmetry across a crowd is the model
working, not a bug to reconcile.

> ⚠ `short` is deliberately **not** a band. `ReachClass` in
> `WeaponProfile` is `short|medium|long`, a banded projection of a
> weapon's authored *length* — a different concept that must never be
> indexed with a band's rank.

### Ask `bandBetween`, not `rangeBetween`

`CombatGraph.rangeBetween` is a pure value-object and falls back to
`close` for a pair it has never seen. That is right for the graph and
wrong for a caller: two people who have not engaged in a 20 m yard are
not in a clinch. **Everything outside the graph asks
`CombatApi.bandBetween`**, which prefers a live edge, else the arena's
cap, else `null` for "not co-present at all".

That fallback is a live trap, and it has bitten twice — once as the
re-seed bug below, once as a splash set that would have swept in every
combatant in the session.

## The arena caps the ladder

Reachable bands derive from the room's **real linear extent in metres**,
never from an authored enum:

| Room extent | Bands afforded |
|---|---|
| ~3 m (the default cell) | `close`, `reach` |
| ~6 m | + `near` |
| ~20 m+ | + `far` |

Thresholds are `combat.range.nearMetres` / `farMetres`. A bar fight
stays a knife fight by physics, and because outdoor cells are already
authored larger, the frontier arms itself.

`Location.getLinearExtent()` is the one number. `CartesianLocation`
returns its own persistent `extent` override, else the zone's
`cellSize`; `SphericalLocation` returns its diameter. A room that
reports nothing gets the conservative melee cap — an unmeasured room
cannot promise the distance a bow needs.

**Extent is deliberately ONE number.** `getSizeScale` (light),
`getVolume` (atmosphere) and `getCeilingHeight` all derive *through* it.
Giving combat its own size field would let a room be small for bands and
large for lux; authoring a bigger room instead makes it dimmer, airier
**and** longer-ranged, because that is what a bigger room is.

## Opening the gap, and closing it

**Every edge opens at the arena's maximum band** — you notice someone
across a room before you are on top of them — **except that an ambush
opens at `close`**. Concealment is what buys the opening band, which is
why a knife-fighter can reach a bowman at all, and it gives stealth a
combat payoff that is not a damage multiplier.

> ⚠ **This changed shipped melee.** The old rule read the two fighters'
> reach ranks, so dagger-vs-dagger began in the clinch. Now a 3 m cell
> opens every pair at `reach`. Reach rank did not stop mattering — it
> moved to where it belongs, deciding who *controls* the gap rather than
> where the fight starts.
>
> The change is mechanically **inert for equal-reach pairs**: `diff` is
> 0, so `range === "reach" ? diff : -diff` is 0 either way. The combat
> gym needed no re-baseline.

`fight advance` and `fight withdraw` step one rung per beat
(`resolveBandStep`), the generalization of the shipped `close` gambit —
which survives as an alias. Two rules the ladder adds:

- **Only an advance is contested.** Reach is a tool for keeping someone
  *out*; a spear-holder has no special power to stop you leaving. The
  contest is also gated to melee, since a spear has nothing to say about
  crossing a bowshot.
- **The arena caps a withdrawal.** You cannot back further away than the
  room is big.

Withdrawal is **per-edge**, so leaving a crowd costs more than leaving a
duel — three attackers is three withdrawals, and "being surrounded is
bad" falls out with no surround mechanic.

## A melee strike cannot cross a ranged band

`reachAdvantage` stays a pure reach term and returns **0 outside melee**.
The cannot-connect fact rides its own `inMeleeBand` predicate at the
strike and counter sites. Overloading the reach number with a large
negative sentinel would have been fewer lines and a lie: a hand weapon
at `near` does not miss because it is out-reached, it misses because
nobody is in melee.

## Resolution — placement, not to-hit

`lib/combat/AimResolution.ts`. A shot resolves from **two public
commitments** and no dice:

- **aim ladder** — `snap` (this beat) → `held` (one beat, visible) →
  `settled` (two beats, target telegraphed)
- **the target's answer** — `stand` · `move` · `cover` · `drop` ·
  `counter`

The pair indexes a 3×5 matrix yielding `miss` / `graze` / `hit` /
`precise`; modifiers then step along that ladder (poor stability,
beyond-envelope, cover quality, target moving).

**A target with no answer takes the worst case** — restrained or
unconscious resolves at `precise`. Honest, grim, and the reason
incapacitation needed its own rung on the consent ladder rather than
being a free tactical win.

**Competence buys tempo and information, never steps.** `resolve` takes
no competence input at all, so there is no channel through which a band
could reach a placement. A novice's shot is exactly as lethal as an
expert's, and that is politically load-bearing.

> **Wave 1 status.** The whole table ships and is unit-tested; the
> engine wires the `snap` row and three answers. `cover` currently
> resolves as `stand` and is pinned by a test that is *supposed* to fail
> when authored cover lands.

## The Delivery Profile

`lib/combat/DeliveryProfile.ts` — computed per (projectile × launcher ×
band), **never stored**:

| Field | What it is |
|---|---|
| `energyJ` | ½mv² from real mass and speed |
| `channel` | the shipped mechanism vocabulary |
| `stability` | flight quality; steps the placement ladder |
| `integrity` | `shatter` / `deform` / `recover` on arrival |
| `payload` | an effect envelope, if any |

**Nothing stores "damage."** What the energy *does* is the response
grid's business. `penetration` is deliberately absent until armor gives
it a consumer.

## Readiness — one field, four families

`lib/combat/EnergySource.ts`. `energySource` answers two questions at
once: where the shot's energy is stored, and therefore whether readiness
holds for free.

| Family | `energySource` | Readiness stored in | Holds free? | Aim |
|---|---|---|---|---|
| Thrown | `muscle` | nothing — instantaneous | n/a | aim *is* the throw |
| Bow | `stored-elastic`, muscle-held | **the archer's body** | **no** — poise/beat | draw **is** aim |
| Crossbow | `stored-elastic`, mech-held | the spanned mechanism | **yes** | decoupled |
| Gun | `chemical` | the cartridge | **yes** | decoupled |

For a gun and a crossbow, aiming and readiness are separate acts; **for
a bow they are the same act**. Two things fall out that would otherwise
need inventing: the crossbow's whole tactical point (held on a doorway
for minutes, paid for in rate of fire), and the gun's real advantage —
**it does not tire you**, which is why firearms won and is the cleanest
balance lever in the design.

Rate of fire is a *consequence* of the same field, never a stored
number: readiness folds into the shot only for a muscle launcher, so a
longbowman needs no separate readiness beat while spanning and loading
stay committed actions.

> Wave 1 exercises only the `muscle` branch. The vocabulary ships whole
> and is tested across all three, because retrofitting an axis across an
> established launcher model later is exactly the expensive move.

## Splash, and the gate over it

Because bands are relationships, a placed effect cannot sit "at a band"
— there is no position to place it at. **Area arrival resolves
relationally: the target, plus whoever is `close` to them.** *"You're
shooting the floor they need"* becomes *"the splash catches whoever is
on top of your target"*, which needs no radius and cannot disagree with
the band model.

Magnitude is **`Dose`**, not a splash rule. The primary catches a share
scaled by placement; each clinched bystander catches a fraction;
whatever is left pools on the floor through the shipped `pour` path. The
split is **volume-conserving** because litres are real, and the dose
curve then does the rest — a `graded` effect scales down honestly on a
bystander, a `threshold` effect may honestly not fire on one at all.

### The consent gate

The gate **cannot** mean "refuse all non-consented harm" — that would
forbid crime itself, and the model deliberately permits attacking the
unwilling (that is what the `consented: false` marker is *for*). The
honest line is **deliberate versus collateral**:

- the **primary** target runs the ordinary terms handshake and is not
  gated; attacking them may be a crime, and the ledger says so;
- every **other** sentient in the splash must already stand under
  consented terms with the thrower, or the whole act is **refused before
  it commits** — nothing thrown, no poise spent, no session opened.

> **Area delivery must not be a cheaper route to a person than aiming at
> them.**

Non-sentients never gate. The gate runs **before** initiation, so a
refused throw cannot be used to probe who is standing where.

## One initiation handshake

`CombatApi.initiate` owns the whole act of starting a fight: reconcile
both sides' standing terms → prompt on conflict → snapshot melee
competence → warm the formation Ideas → read the ambush *before*
revealing the attacker → open / join / merge.

Every initiating verb goes through it. `throw ... at` routing
"identically" to `attack` is a fact only while there is one copy of the
sequence. `onConflict` stays a caller-supplied callback because asking a
player a question is a UI act: the engine decides *that* a conflict must
be answered, the controller decides *how* to ask.

## The two adopted seams

- **`HazardDelivery.range`** gains `'ranged'`. A ranged delivery has
  already found its victim when it resolved, so the traversal trigger
  skips one: walking through where a flask broke is not what springs it.
- **`MagicLogic.deliverAt`** gains a band gate on top of the same-scene
  check (which stays — cross-room fire is out of scope, and that is what
  makes an exit a real escape). The envelope defaults to
  `magic.spellEnvelope`, seeded `far`, so **every shipped spell behaves
  exactly as before**.

**`DischargeOptions.origin`** is the detail that makes thrown carriers
work at all. Reachability measures from the *origin*, and a `Material`
singleton has no place of its own — so without it a thrown flask's
payload would issue from the **thrower** and a `close`-envelope contact
effect would refuse across the gap it had just crossed.

## `throw`

`cmd/inventory/throw.yaml` + `obj/command/inventory/ThrowController.ts`,
afforded by **`TangibleMixin`** — the thing with **mass** confers the
verb, outward on whoever holds it (`environment`) and sideways on
whoever stands where it lies (`peers`).

Not `ContainerMixin`, which was the first cut and was wrong twice over:
"I can hold things" is the right home for `drop`/`give` and says nothing
about whether what you are holding can be thrown — and **a room is a
Container**. Mass is the honest predicate, and it is also the ½mv²
input, so the mixin that affords the verb is the one that supplies its
physics.

Throwing still operates through **containment** — the category names the
substrate, not the drama, the same reasoning that put `disarm` under
`device` and `drink` under `bulk`. That is what makes "works outside
combat" free: being a Combatant is not required.

**You must be holding it, and that is the point.** The item argument
resolves over `reachable`, deliberately wider than `inventory`, so a
rock on the ground *matches* — and then `mustBeHeld` refuses with
*"you'll have to pick up the rock first"*. Folding the pickup into the
throw would have been fewer keystrokes and would have deleted a tell:
bending down and taking hold of a rock is a **public act**, and in a
world where that rock is about to become a weapon, the bystanders are
owed the beat of warning. Narrow the scope back to `inventory` and the
verb simply reports no such rock; the refusal, and the lesson in it,
disappear. Pinned by
`api/__tests__/command-binder-throw.test.ts`.

- `throw <item>` — a ballistic relocation. No target, no gate, no fight.
- `throw <item> at <someone>` — an initiation, through the same door as
  `attack`.

The two parses are told apart by the `at` binding, not by a second verb.

**A potion only acts if its `route` is `contact`.** Throwing an `oral`
draught wastes the flask — which is the rule that kills the
throw-everything case without anyone having to write a rule.

## Content

| Thing | Where | Why it exists |
|---|---|---|
| blistering draught | `content/obj/material/potion/` | `route: contact` — the potion that earns its throw |
| flask of blistering | `seeds/obj/items/` | a glass vessel, so it breaks |
| the long meadow | `seeds/domain/newbie-wilds/crossroads/` | `extent: 12` — the first room affording `near` |

**Glass gained `hardness: 550` / `toughness: 0.5`.** It authored neither
before, so a flask derived `recover` and would never have broken —
the whole feature silently inert. Against steel's 600/200 that ratio
*is* brittleness: nearly as hard, almost none of the toughness. The
values are honest, not tuned to make a flask break.

## Two bugs this build found in shipped code

**`CombatLogic.ts` re-seed (fixed).** `rangeBetween(...) === "close"` was
serving as a "have these two met yet" sentinel, asked immediately *after*
`addEdge` minted the edge at the `close` default — so it could not tell a
genuinely-clinched pair from a fresh one. A pair that had fought its way
inside was reset to its opening band every time the engine re-picked the
target, silently, because `close` is also a legal band. Now
`pairHasRange`, asked before the mint and reading both directions.

**`AttackController` has no tests.** The plan called for a byte-parity
regression over them; there are none, and never have been. The
extraction is pinned instead at `CombatApi.initiate`, where the logic now
lives. The controller-coverage gap is
[antipatterns.md](../antipatterns.md)'s, not this build's, but it is
worth knowing it is still open.

## What Wave 1 actually closes

The requirements carry **56 acceptance criteria**, and Wave 1 is one of
four waves — so most of them are open by design, not by omission. Fully
closed: the four-band ladder and its per-pair symmetry (AC 1, 2), the
arena cap and the per-location extent behind it (3, 4), band stepping
and its cap refusal (5), the pure `DeliveryProfile` and the
carrier-blindness it buys (14), the aim×answer matrix (8), splash and
its consent gate (24, 25, 31), the ranged hazard route (26), and the
`throw`-is-an-`attack` handshake (47).

Two worth stating precisely rather than claiming loosely: **AC 31** is
proven through `throw` only, because no shooting verb exists yet; and
**AC 52** lands both halves in code but its archer case cannot be tested
until W3 mints an archer.

Known misses, called out because the plan did not list them: **AC 53**
(the N=8/N=16 large-fight profile) was never recorded, and **AC 55** is
five of eight sibling docs — `crafting.md`, `chattel.md` and
`concealment.md` are untouched because Wave 1 opens no seam in them
(recoverable projectiles and pattern-key registration are W3/W4 work).

## What the live drive reached, and what it did not

Driven in a browser against a fresh database, Wave 1's untargeted throw
works end to end: clone a flask, drop it, `throw flask` refuses with
*"you'll have to pick up a sealed glass flask first"*, `get` it, and
`throw flask` hurls it. That closes the untargeted half of AC 46.

**The band-ladder legs did not run.** The plan (§7.4) assumed the
shipped `goto` would carry a driver to the long meadow, and it does
not: `goto`'s argument scope is `["online", "/obj/**", "reachable"]`,
with no `/domain/**` — and every room in the world lives under
`/domain/`. So the one author verb for "take me to that path" cannot
address content, and `goto /domain/newbie-wilds/crossroads/longmeadow`
answers *"no match"*. The fix looks like one scope entry, but it is a
behaviour change to a shipped author verb and is deliberately **not**
made here. Until it lands, reach the meadow by walking or by TPA.

Still undriven, therefore: the targeted throw across a real band gap,
splash over a clinched set, the consent refusal, and the band ladder
itself (AC 23, 24, 25, 47, and the rest of 46).

⚠ A second bootstrap blocked the drive before that one, and it is worth
recording because it is not what anyone guesses: `goto` and `clone` do
**not** consult `isWizard`. That axis is code-trust (eval / reload /
source writes). They call `AccessApi.can`, which resolves untitled
content to the `core` group — and `core` is minted empty, has no env
seed, and is owned by `'system'`, so `group add` refuses everyone. A
fresh database has nobody who can author anything.
`packages/server/scripts/dev-grant-core.mjs` is the stopgap; it is
meant to die with the `core` group.

## Deliberately out of scope

- **Cross-room fire.** Sight may cross a vista; combat may not. This is
  load-bearing rather than a limitation: it is what makes leaving
  through an exit a genuine escape.
- **Coordinates, derived position, intra-room geometry.** Bands are
  per-pair relationships. "Can I see you in here" is concealment, which
  is shipped.
- **Energy weapons**, parked on the electricity substrate.

## The roadmap

| Wave | Covers |
|---|---|
| **W2 — cover & armor** | authored cover (directional, destructible, capacity-leased) · overturnable furnishings · armor on the response grid, point→blunt conversion · suppression / held aim · formation band preference · the readout ladder + cross-reading split · the lingering residue hazard · the `vapour` route · `DeliveryProfile.penetration` |
| **W3 — bows, crossbows, less-lethal, acoustics** | graded archery fit · readiness as committed actions + the bow's hold window and dry-fire · `elasticity` as a material property · the incapacitation rung + the less-lethal family · per-metre sound attenuation · the four NPC doctrines + morale + NPC ammunition |
| **W4 — guns** | the field model · reliability-vs-output degradation · the generalized fast-wear axis · components · pattern keys · grade buys reliability · catastrophic failure on the readable-state rail · registration as the chattel ledger · the negligent-discharge leg |
| **Content** | the Practicum range · the armory · guard patrol density · the accessory catalogue · the installed launch regime |

## Cross-references

- [combat.md](./combat.md) — sessions, poise, gambits, terms, the graph
- [materials-response.md](./materials-response.md) — the `Channel`
  vocabulary and the response grid
- [hazard.md](./hazard.md) — `HazardDelivery`, the `ranged` seam
- [magic.md](./magic.md) — the effect union, `deliverAt`
- [concealment.md](./concealment.md) — what an ambush buys
- [location.md](./location.md) / [zone.md](./zone.md) — room extent
- [chattel.md](./chattel.md) — spent-projectile ownership (W2+)
- [command-spec.md](./command-spec.md) — the `inventory` category
