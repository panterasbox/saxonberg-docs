# Blood — type, draw, store, transfuse

The clinical-medicine build's blood vertical: a body's blood is a
**heritable type** it can learn, a **drawable unit** it can store until it
spoils, and a **transfusable good** whose incompatibility is a real,
graded harm. Built on the shipped `bloodVolume` vital; see
[harm.md](./harm.md) and [vitals.md](./vitals.md) for the substrate.

## The type

- **`Species.bloodGroups`** (`{ alleles: Record<string, number> } | null`)
  — an ABO allele-frequency table (`A`/`B`/`O`). `null` = the engine
  default *one allele `O` at frequency 1*: every member of an unauthored
  species is `O` and compatible within its species. Data, not a guard — a
  wolf has blood and one type until an author says otherwise. `spoiler: 1`
  (learned by testing, not printed in a field guide). The frequencies are
  pure fiction; the **ABO pattern** is the transferable clinical judgment.
- **`VitalsMixin.bloodGenotype`** (`string | null`) — an author PIN
  (`"AO"`); `null` derives deterministically. **`bloodTyped`** — has
  anyone tested it. The genotype is **seeded on identity**
  (`Seeded.unit` over an FNV fold of `getIdentityPath()`), never
  `Math.random`, so an unrolled body and a rolled one always agree and
  nothing need persist for the ordinary case. Both fields join
  `forkSlice_Vitals` (a corpse carries its type — the fork emits the
  RESOLVED genotype, so an identity re-roll can never disagree).
- **`lib/vitals/BloodType.ts`** — a phenotype value object (`speciesPath`
  + ABO, plus `mixed`). Instance methods only (`lint:lib-statics` is at
  ceiling): `isCompatibleDonorFor` (O→any, A→A/AB, B→B/AB, AB→AB, same
  species) and `mismatchFor` (`0` ok · `1` ABO · `2` cross-species).
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

## Deferred → `blood-slate`

Contamination/screening of a unit (the `ContaminableMixin` reading the
same payload), the shortage summons, gift-only credits, Rh, the
inter-species compatibility topology (v1 is flat; `mismatchFor` is where a
graph goes), the consent prompt on a conscious recipient, heart-rate
compensation. `BulkPayload.blood` and `receiveBlood` are the seams.

## History

Shipped by the clinical-medicine build (W1/W5). See
`docs/plans/clinical-medicine-plan.md` D2/D3/D4/D11. **Review round
(2026-09-24):** `bleed`/`transfuse` became durative engaged acts (the
`DRAW_DURATION_S`/`TRANSFUSE_DURATION_S` dials on `BLOOD_DEFAULTS`) — the
time cost is the honest friction on donation; the completion effect and
the synchronous gates are unchanged.
