# Blood — type, draw, store, transfuse

The clinical-medicine build's blood vertical: a body's blood is a
**heritable type** it can learn, a **drawable unit** it can store until it
spoils, and a **transfusable good** whose incompatibility is a real,
graded harm. Built on the shipped `bloodVolume` vital; see
[harm.md](./harm.md) and [vitals.md](./vitals.md) for the substrate.

## The type

- **`Species.bloodGroups`** (`{ alleles: Record<string, number>;
  system?: string } | null`) — an ABO allele-frequency table (`A`/`B`/`O`)
  plus (blood-economy build D3) an optional **blood `system`**. `null` =
  the engine default *one allele `O` at frequency 1*. Data, not a guard — a
  wolf has blood and one type until an author says otherwise. `spoiler: 1`
  (learned by testing, not printed in a field guide). The frequencies are
  pure fiction; the **ABO pattern** is the transferable clinical judgment.
  - ⭐ **`system` (D3)**: compatibility is of the BLOOD, not the taxon. Two
    bodies are ABO-comparable iff their `system` matches; unauthored = the
    species' own path (the flat per-species default every shipped species
    keeps). Authoring the same `system` on several species lets their
    members cross-transfuse — the `homo/*` rows ship three clusters
    (`hominid`/`fae`/`giant`) so a dwarf can take a human's blood and an
    elf cannot. See `instrumentation.md` (`analyze blood`).
- **`VitalsMixin.bloodGenotype`** (`string | null`) — an author PIN
  (`"AO"`); `null` derives deterministically. **`bloodTyped`** — has
  anyone tested it. The genotype is **seeded on identity**
  (`Seeded.unit` over an FNV fold of `getIdentityPath()`), never
  `Math.random`, so an unrolled body and a rolled one always agree and
  nothing need persist for the ordinary case. Both fields join
  `forkSlice_Vitals` (a corpse carries its type — the fork emits the
  RESOLVED genotype, so an identity re-roll can never disagree).
- **`lib/vitals/BloodType.ts`** — a phenotype value object (`system` +
  ABO, plus `mixed`; the field was renamed from `speciesPath` in D3).
  Instance methods only (`lint:lib-statics` is at ceiling):
  `isCompatibleDonorFor` (O→any, A→A/AB, B→B/AB, AB→AB, same SYSTEM) and
  `mismatchFor` (`0` ok · `1` ABO · `2` cross-SYSTEM). `BloodUnit` carries
  `system` (stamped at `drawBlood`; `speciesPath` kept for provenance);
  `VitalsMixin.bloodSystemOf()` resolves it (the species' declared system,
  else its own path).
- `VitalsMixin.bloodType()` reads the phenotype (`null` for a bloodless
  clade, gated on `hasVitalSign('bloodVolume')`); `isBloodTyped()` /
  `markBloodTyped()`.

## The unit

A drawn unit is a **perishable Material** — `/stuff/idea/material/tissue/
blood` — in any bulk holder; **no `FreshnessMixin` widening**. What
cannot derive from the Material rides `BulkPayload.blood`
(`lib/vitals/Blood.ts`, the per-module declaration-merge pattern):
`{ speciesPath, type, labelled, donorIdentityPath }`. ⚠ **`type` is the
TRUE phenotype ALWAYS** — an unlabelled unit still reacts if it truly
mismatches; `labelled` is the epistemic half (whether anyone tested it),
which is the only thing that lets a competent transfuser SEE a mismatch
before making it. Shelf life is tuned against the shipped `Freshness`
arithmetic (~3 game-days at 293 K, ~4 game-weeks in a cold larder), so
warm blood can't be hoarded but cold blood buys a hunt.

The pour-blend is wired into `BulkableLogic.transfer` beside
freshness/cure/pathogens: two units of the same labelled type stay that
type; anything else becomes `mixed` — compatible with nobody, which stops
the decant-to-launder trick.

## The acts (D5, `trade-medicine`)

The **`Syringe`** (`capability: phlebotomy`) affords all three:

- **`test [patient]`** — reads + labels the type.
- **`bleed [donor] into <vessel>`** (alias `venesect`) —
  `Vitals.drawBlood(litres)`: spends `bloodVolume` and the **`marrow`**
  reserve (a tenth biological reserve; `bleed` refuses below the donation
  floor); fills the vessel with a stamped unit. Refuses an unconscious /
  low-volume / wan donor, or a vessel already holding another type. ⭐ The
  draw is a **LONG durative engaged act** (`BLOOD_DEFAULTS.DRAW_DURATION_S`,
  a `ManualBuildStep` on the `'hands'` slot): the real barrier to giving
  blood is the TIME you sit there, not the volume — the `marrow` reserve
  models the recovery cost, the engagement the ACT cost. All the gates run
  synchronously at dispatch; the unit is drawn at COMPLETION, so a barge-in
  aborts it and no blood is taken. (⚠ Multi-word vessel names are quoted:
  `bleed into "blood bag"`; every object arg is single-token.)
- **`transfuse <patient> from <vessel>`** — a **SHORT interruptible**
  engaged act (`TRANSFUSE_DURATION_S`) — the field-medic-under-fire tension;
  an abort gives nothing. blood → the unit, `salt-water`
  → a saline expander, else refused; spoiled refused. ⭐ **The judgement:**
  a `competent+` giver (in `max(nursing, medicine)`) who can SEE a
  labelled mismatch on a typed patient **refuses** — competence buys
  judgement, never a better transfusion. `Vitals.receiveBlood`: compatible
  closes the gap to baseline; saline caps at the shipped `0.85` plasma
  ceiling; incompatible delivers only the plasma fraction and inflicts the
  graded, decaying **transfusion-reaction** Condition (×2 cross-species),
  which drains into the shipped dehydration→dying cascade — nothing new
  kills.

## Autarky

Every consumable is self-sourceable: your own body is the donor, the
`marrow` reserve regrows on the metabolism slice (nutrition-gated, ~2
game-weeks a unit — tight, not free), `salt-water` is the saline floor, a
blood bag is a bought or made Receptacle. Own blood is always compatible
(same identity → same seeded genotype).

## The blood economy (blood-economy build)

A civic bank of donated, typed, perishable units with an un-withdrawable
floor, over shipped substrate. All in `trade-medicine` + `terminus`; the
kernel grew five small seams.

- **The transfusion harm row (D5)** — a transfusion just WORKS; there is
  no consent step (nobody rationally refuses life-saving blood, and
  registration was a non-choice — both were cut from the shipped build,
  see History). The one thing the ledger records is HARM: forcing a
  REACTING unit (`reaction > 0`) into ANOTHER body appends a `harm` row,
  `consented: false` — the accountability trap's producer shape, so bad
  blood given as a weapon is a crime on the record. A compatible unit
  records nothing; self-use never records.
- **`DonationBankMixin` (kernel `lib/commerce`, D2)** — a bank read BY LOT
  over a configured vault (`_vaultPaths`), ignoring the door (a closed
  sealable reads empty through the perception sheet; the registrar knows
  her own fridge). `getLots`/`shortLots`/`parLevel`/`shortfall`,
  `takeUnit`/`takeCompatibleUnitFor` (FIFO), `receiveGift`; custody deeds
  on persons. The one blood-aware line is `lotKeyOf` (the ABO type).
  Composed ONLY by the pack class `BloodWindow = DonationBankMixin(Tariff)`.
- **The fee (D6)** — a `transfusion` `SERVICE_KIND`; `order transfusion`
  draws a compatible unit and `collect`s (self-first: the fee attributes
  to the window's own house, not the practice sharing the ward — two
  houses in one room). The floor is on-the-house (no debt object); nobody
  dies for an empty purse. The DONOR is never paid (gift-only).
- **The gift credit (D8/D9/D10)** — a PLAYER giving their OWN blood earns
  a chronicle deed, a disposition signature (generosity always, compassion
  when the lot was short — the first live authored `dispositionValence`,
  grafted onto `AdvancementMixin.creditSignature`), and a renown
  `applaud` reaction folded by a scheduled recompute. An NPC's gift is
  plumbing; giving another's unit earns nothing.
- **The readings (D15)** — `analyze blood` (the compatibility ladder,
  crossing species by system) and `analyze bank` (the typed panel at
  untrained + the custody trail at expert).
- **Supply (D13)** — a producer floor (`Stock` + the spawn sweep) +
  the `supplies` runner brain (floor→fridge through the door dance) + a
  visible donor Extra; the `banks` registrar brain (the floor transfusion,
  the room summons — pure pull, no `tell` — and the ticker notice via
  `BusinessEntity`'s `PublisherMixin`).
- **The ghost sign (D12)** — a dead Goodkin `NeonSign` (intensity 0): the
  Paramount Decree made literal. The Terminus window is PRIVATELY, NPC-run
  (no player-fillable seat in v1) under the old Goodkin name; the corpo
  kept collection + carting (the Bloodworks upstream), divested the
  giving-away. See `corpo.md`, `employment.md`, `retail.md`.

## Deferred → `blood-slate`

Contamination/screening of a unit (the `ContaminableMixin` reading the
same payload), Rh (a second axis — the lot vocabulary is a string),
player operation of the window (the seat gate is already right; only the
roster keeps players out), the paid-donor lever (the gift gate is where a
sale would sever the credit), a receivable for the floor fee
(→ `credit-slate`), epoch-gated typing, and **post-mortem body/organ
donation** — a standing directive that DOES have a decision matrix (death,
re-embodiment, and the corpse are all real), which the cut donor card did
not. `BulkPayload.blood`, `receiveBlood`, and `DonationBankMixin` are the
seams.

## History

Shipped by the clinical-medicine build (W1/W5). See
`docs/plans/clinical-medicine-plan.md` D2/D3/D4/D11. **Review round
(2026-09-24):** `bleed`/`transfuse` became durative engaged acts (the
`DRAW_DURATION_S`/`TRANSFUSE_DURATION_S` dials on `BLOOD_DEFAULTS`) — the
time cost is the honest friction on donation; the completion effect and
the synchronous gates are unchanged.

**Blood-economy build (2026-10-04):** the full bank economy —
`DonationBankMixin` + the `BloodWindow`, the `transfusion` service +
two-houses-in-one-room, the gift credit (the disposition graft + renown),
`analyze blood`/`analyze bank`, the three brains, and the Terminus Goodkin
window under its dead sign. D3 replaced flat cross-species incompatibility
with a declared blood `system`. See `docs/plans/blood-economy-plan.md`.

**Donor-card cut (2026-10-05):** the standing donor card and its
consent ladder (`PersonaMixin.donorCard`, `transfusionConsent`, the
`donor` verb, the donor roll on `analyze bank`) were **cut** before merge.
They modelled a decision that is not one — registration was free and
non-binding, and refusing life-saving blood is near-always irrational, so
neither half was a weighable choice (lens 4: a gauge must convert an
undecidable choice into a calculable one; this converted a non-choice into
a permissions system). What remains is the gift loop, the supply, the
window, the readings, and the harm-on-mismatch row above. Post-mortem
body/organ donation — which DOES have a matrix — is deferred to
`blood-slate`.
