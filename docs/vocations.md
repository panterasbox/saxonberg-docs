# Vocations & industries — the register

**A living inventory of the work the world supports**, plus the methods
for finding the work it *doesn't* yet.

> **This is reference material, not a slate.** It spans every subsystem
> and it is never "finished" — entries move between statuses as builds
> land, and the **gap** column is the point.

Started 2026-07-31, seeded from the transport/industry design run.

## ⭐⭐⭐ The governing test

> **A vocation exists iff there is unmet demand — somebody needs
> something they cannot do themselves.**

The register is a **demand audit, not a wishlist.** Otherwise it fills
with thematically pleasing roles nobody ever hires. Every vocation
found in the seeding session passes the test: people need goods moved,
stored, broken down, and sold when the price is unknown.

## What makes a vocation real — five criteria

| # | Criterion | Why it bites |
|---|---|---|
| 1 | **Unmet demand** | somebody needs it and **cannot self-serve** |
| 2 | ⭐⭐ **A gated capability** | competence, an instrument, a licence, or access. **If anyone can do it, it is a chore, not a job.** |
| 3 | **A repeatable loop** | not a one-off errand |
| 4 | ⭐ **Income that is PAID, not MINTED** | a vocation funded by the central bank is a **subsidy**. Real work is paid by another player or by an NPC business with an actual budget. |
| 5 | **A failure mode** | you can do it *badly*, or it is not a skill |
| ⚠ | ⭐⭐ **MOST VOCATIONS ARE LADDERS — only the upper rungs qualify** | the bottom of every trade is a **chore you do for wages**. The **teller**, the **ceremonial witness** and the **stable-hand** are *texture under* a real vocation, never entries beside it. A brake on the register inflating job titles into professions. |
| ⚠ | ⭐⭐ **NEVER INVENT A NEED TO CREATE A MARKET** | if a vocation requires a **new player obligation** in order to exist, **the demand was fabricated** — criterion 1 failing in disguise. (The **innkeeper** is the worked example: justifying an inn with a sleep *requirement* is manufactured demand; the honest justification is that **NPC schedules make the inn the place that stays open**.) |

**(2) is what turns an activity into a livelihood** — and it is exactly
the instrumentation thesis.
**(4) is the one violated by accident.**

## ⭐⭐ Four gap-finding methods, ranked

### 1. The chain walk

For each good: **extract → process → move → store → sell → use →
maintain → dispose.** Unstaffed links are gaps.

> **This produced the entire seeding session.** Walking a steer from
> pasture to butcher generated freight, the depot, the salvage yard and
> the second-hand market.

Proven, and the first thing to run against **farming** and **mining**.

### 2. ⭐⭐ The substrate audit

Walk the shipped subsystem list and ask **"who does this for a
living?"** Systematic, bounded — and it doubles as a health check:

> **A subsystem with no vocation attached is usually half-grown.**
> Nobody makes a living from it, which usually means nobody *needs* it.

Electricity has a linesman; harm is getting a medic. The audit's first
run found **five empties — and four of them answered themselves:**

| Empty substrate | Candidate found |
|---|---|
| **thermal** | ⭐⭐ **the icehouse keeper** — and it is what makes refrigeration possible *before machines* |
| **weather** | ⭐⭐ **the almanac-maker** — farmers need forecasts, instrumentation gates it |
| **concealment** | ⭐ **the customs officer / searcher** — the one who finds what is hidden |
| **chronicle** | ⭐ **the registrar**, and **the monument mason** at the necropolis |
| **belief** | ⭐ **the herald** — identity verification as a trade (weakest of the five; keep looking) |

> **That the method answers its own questions is the argument for
> running it on a schedule.**

### 3. The external sector checklist

Run a real-economy taxonomy against the world. **The only method that
catches blind spots**, because freeform brainstorming can only surface
what you have already thought about.

### 4. The demand audit

Start from **"what does a player need that no vocation provides?"**
This is what surfaced the **appraiser**.

### ⭐⭐⭐⭐ And the disposal technique: DECOMPOSE, never accept or reject wholesale

> **When a real-world vocation does not fit, decompose it — the parts
> land in different places.**

The **notary** is the worked example, and the register's **first honest
NO**: its *attestation* is already free (the kernel knows who acted; the
record witnesses everything; append-only means non-repudiation), so the
stamp-and-seal role fails **criterion 2**. But decomposing it yields
**the scrivener** (a real vocation — clauses are machine-enforceable, so
drafting has a real failure mode), **full faith and credit** (a
constitutional question, not a job), and **ceremony as deliberate
friction** on irreversible acts.

**A register that only adds roles is a wishlist. One that can say no is
a tool.**

## The register

Per entry: **what gates it**, **who pays**, **which subsystem it rides**.

### Primary — extract

| Vocation | Status | Gate | Paid by |
|---|---|---|---|
| **miner** | **shipped** ([mining](./subsystems/mining.md)) | skill + tools | the smelter, through the assay shed — ⭐ *or nobody: with an adit and no shaft, a claim-holder walks their own ore out and owes the co-op nothing* |
| farmer | **in build** | land + skill | produce buyers |
| rancher | **in build** | land + stock | livestock buyers |
| fisher | designed (fishing) | tackle + skill | market |
| **prospector** | **shipped** ([mining](./subsystems/mining.md)) | **assay** (instrument + competence) | speculative — ⭐⭐ *and the survey record itself is the asset: a per-viewer DISCOVERY belief you can trade, which is why negative knowledge still sells* |

### Secondary — process

| Vocation | Status | Gate | Paid by |
|---|---|---|---|
| crafter (many branches) | **shipped** | Discipline + tools | buyers |
| bartender / cook | **shipped** | on-shift `MakerMixin` | the house |
| slaughterer / butcher | designed | premises + skill | trade |
| **victualler** (preserving) | designed (preservation · hearth & larder) | premises + competence | trade — ⚠ *the vocation the icebox eventually devalues; that is the point* |
| **mana refiner** | designed (mana economy) | premises (industrial zoning) + cheap power | trade — ⭐ *concentration, never creation; needs the energy economy as its input* |
| **salvager** | designed | **assay** + yard | materials buyers |
| **smelter** | **shipped** ([mining](./subsystems/mining.md)) | premises (industrial zoning) + charcoal | ingot buyers — ⭐ *buys its ore out of REVENUE, which is what removed the deferred CB-lending dependency* |
| miller / tanner | **GAP** | premises (industrial zoning) | trade |

### Tertiary — move, store, sell

| Vocation | Status | Gate | Paid by |
|---|---|---|---|
| teamster / drover | designed (freight) | driving + a rig | shippers |
| carrier (post) | designed (delivery) | employment | the provider |
| warehouseman | designed | premises + **bailee duty** | depositors |
| **forwarder** | designed | *no premises* — pure broker | shippers |
| shopkeeper | **shipped** | premises + stock | customers |
| pawnbroker | designed | capital + licence | borrowers |
| auctioneer | designed (auction) | venue + **reputation** | commission |
| **broker / trader** | **GAP** | information | spread |
| **loan officer / credit assessor** | designed (insurance § banking) | **judgment** | the bank |
| **credit surveyor** | designed | goes and looks | subscribers |
| **rating publisher** | designed (insurance § ratings) | **a method + a name** | subscribers (⭐ *timeliness is the product*) |
| **money changer** | **GAP** | **assaying coin** (clipping, debasement) | the spread |
| **accountant** | designed (insurance § accountant) | **accrual judgment** | businesses (⭐ *demand comes from the banker*) |
| **auditor** | designed | independence + competence | ⚠ **must not be the same firm as the accountant** |
| **forensic accountant** | designed | competence | ⭐ *the route to the untouchable — they got Capone on tax* |
| **revenue inspector** | **GAP** | office | the treasury (blocked with the fiscal cycle) |
| **collector** | **GAP** | a court order | creditors |
| **teller** | **GAP** | employment (attendant shape) | the bank |

### Services

| Vocation | Status | Gate | Paid by |
|---|---|---|---|
| medic | **next build** (mortality) | Discipline | patients |
| journalist | designed (press) | credibility | readers / patrons |
| **appraiser** | **GAP** | **certified instrument** | fee per assay |
| navigator / cartographer | designed | maps + competence | travellers |
| guard / escort | designed | combat | caravans |
| advocate | designed (cooperative) | competence | clients / the state |
| **teacher** | **GAP — see below** | competence in the subject | students / institution |
| **veterinarian** | **designed** (insurance § the vet) | Discipline + **observation without self-report** | stock owners / companions — ⭐ *the only vocation that can shut the freight network* |
| **meat inspector** | designed | certified competence | the locality — *condemnation is a `directive`* |
| **farrier** | **GAP** | ⚠ **a crafting branch, not medicine** | horse owners |
| **innkeeper** | **designed** — *condition met 2026-07-31* (npc-behavior § schedules) | capital + reputation (**not craft**) | travellers |
| **ostler** (master, posting stable) | **designed** (insurance § the ostler) | **husbandry** + fleet judgment | travellers — ⭐ *road capacity is stable capacity* |
| *stable-hand* | — | *wage* | ⚠ **texture, not a vocation** — the bottom rung of the ostler's ladder |
| ~~notary~~ | ⭐ **REJECTED 2026-07-31** | — | **the substrate already attests** — the register's first honest NO (insurance § the notary) |
| **scrivener** | designed | **drafting judgment** | parties to an instrument — *a badly drafted clause executes exactly as written* |
| **insurer** | **designed** (insurance) | capital + portfolio construction | premiums |
| **barber / tailor** | **GAP** | skill | customers |
| **steward** (household / land) | **designed** (household) | **Stewardship Discipline** (the causal read) + capacity | householders — **trade** or **wage**; ⭐ *the product is attention while you are elsewhere* |
| *houseworker / cleaner* | — | *wage* | ⚠ **texture, not a vocation** — anyone can, and the mess freezes while you are away |
| **nanny / carer** | ⛔ **BLOCKED** | — | **no dependents exist**, deliberately — see below |
| ~~homemaker~~ | ⭐ **REJECTED 2026-08-06** | — | **universal demand meets universal self-service** — the register's second honest NO (below) |

### ⭐⭐ Four payment shapes, not two

Criterion 4 needs a finer grain than *paid vs. minted*, because a whole
column of this register is paid differently:

| Shape | Paid by | Example |
|---|---|---|
| **trade** | customers, per transaction | shopkeeper, appraiser |
| **wage** | an employer with a budget | roadmender, gatekeeper |
| **treasury** | the polity, from tax and appropriation | constable, inspector, gaoler |
| **duty (unpaid)** | nobody — it is an *obligation*, not a job | **jury service** |

The fourth exists so the register stays honest: **a duty is not a
vocation.** Sortition jury service is a civic obligation and it must
never be listed as work.

### ⭐⭐⭐ And the treasury column is BLOCKED on one unbuilt system

Every public servant below — constable, inspector, gaoler, firefighter,
clerk, public teacher — is paid **from the treasury**, which requires
**tax → budget → appropriate → disburse** (Art. VIII §4).

> **The fiscal cycle is not one of four government gaps. It is the gap
> that unlocks an entire class of work.**

Until it exists, public service can only be modelled as **offices held**
(founder-default, unpaid) rather than **posts worked**.

### ⭐ Office vs. post — the distinction the substrate already makes

- an **office** is *held*: singular, accountable, with a named holder
  and a founder default ([governance.md](./subsystems/governance.md)'s
  five `OFFICE_APPARATUS` seats);
- a **post** is *worked*: a `Position` on a Business, many holders,
  shifts and wages ([employment.md](./subsystems/employment.md)).

**The commissioner is an office; the constable is a post.** Getting this
right per row is most of the civic design.

### Civic & public service

| Vocation | Status | Kind | Gate | Notes |
|---|---|---|---|---|
| Prime Minister · Speakers · CB Governor | **shipped** | office | confidence / election | the five `OFFICE_APPARATUS` seats |
| constable | designed (policing) | post | training + warrant | *most police work is order and service, not crime* |
| **detective / investigator** | designed (deduction) | post | competence | *"check the yard's book"* is its shipped-substrate act |
| commissioner | designed | **office** | appointment | the charter → policy hook |
| **gaoler / warden** | designed (prison) | post | licence | the custody book |
| **bailiff** | **GAP** | post | warrant | court enforcement, distinct from policing |
| judge / juror | designed (cooperative) | **duty** | sortition + tenure | ⚠ **a duty, not a vocation** |
| advocate / public defender | designed | post or trade | competence | client-paid *or* treasury-paid |
| **court clerk / registrar of the Roll** | **GAP** | post | licence | somebody maintains the Roll and the docket |
| inspector (roads, scrap, weights) | designed | post | certified instrument | **the `directive` instrument forces this role into existence** |
| **weighmaster** | **GAP** | post | certified instrument | named in the depot design |
| **customs officer** | **GAP** | post | warrant | the Gate; tariffs; ⭐ **and the vocation of the concealment substrate — the one who finds what is hidden** |
| **firefighter** | **designed 2026-07-31** | post (+ volunteer, + bucket brigade) | training | fire § The fire service — *fires matter because they SPREAD*; prevention is the gameplay; **a burning street is district-scoped content anyone can join** |
| **fire investigator** | **GAP** | competence | the coroner's sibling — arson + insurance fraud |
| **insurer / underwriter** | **designed 2026-07-31** | capital + **portfolio construction** | insurance — *the book is a map, concentration is the risk* |
| **adjuster** | designed | competence | claims; the fire investigator's sibling |
| **surveyor (insurance)** | designed | certified instrument | ⭐ **the insurer is a private regulator** — prevention becomes a discount |
| **sanitarian / public health** | **GAP** | post | competence | quarantine, contagion; the disease thread |
| **coroner** | **GAP** | office | competence | ⭐ `causeOfDeath` is a **stamped field** — somebody should *determine* it. Feeds the accountability ledger. |
| **registrar (births, deaths, names)** | **GAP** | post | licence | ⭐ the **chronicle**'s vocation |
| **archivist / librarian** | **GAP** | post | competence | the integrity branch's human face; the Museum |
| **assessor / tax collector** | **GAP** | post | warrant | blocked with the fiscal cycle |
| **postmaster** | designed (delivery) | post | the provider | the post office you can walk into |
| **harbourmaster** | **GAP** | office | appointment | the Gate's port |
| **surveyor (Lands & Works)** | **GAP** | post | instrument | the office already exists in the Civic Quarter |
| **relief administrator** | **GAP** | post | office | the WPA floor + match |
| soldier / militia | designed | post | muster | Garrison; the war faces outward |
| **crier** | **GAP** | trade | none | notice; the pre-literate publication |

### Industry & the trades

| Vocation | Status | Gate | Notes |
|---|---|---|---|
| **wainwright / wheelwright** | **GAP** | Discipline | named by freight; makes the rigs |
| **cooper** | **GAP** | Discipline | barrels — the bulk-goods container |
| **stevedore / docker** | **GAP** | employment | ⭐ **Wharfside already has a dockers' hall**; the labour half of the depot |
| **icehouse keeper / ice cutter** | **GAP** | premises + thermal read | ⭐⭐ **THIS IS THERMAL'S MISSING VOCATION** — and it is what makes the refrigeration arc possible before machines |
| miller · tanner | **GAP** | premises (industrial zoning) | the classic nuisance trades (⭐ the smelter SHIPPED — see § Secondary) |
| **charcoal burner / collier** | **shipped** ([mining](./subsystems/mining.md)) | skill | ⭐⭐ *a judgment craft with a real downside — too much air and the charge goes to ash, too little and you draw half-burnt brands, and you can lose a whole burn.* Fuel for everything above, and the mine's timber comes off the same stand |
| **gunsmith · powder maker** | designed (ranged) | licence | credential-gated |
| **shipwright · ship-breaker** | **GAP** | premises | the breaker is a LULU |
| **quarrier** | **GAP** | land + tools | mining's sibling |
| **almanac-maker / weather-reader** | **designed** (insurance § the almanac-maker) | ⭐ **a TIME SERIES** (years of records — not skill or capital) | farmers — *a tighter planting window*; ⭐⭐ **owns DATA where the ratings agency owns a METHOD** |
| **herald / identifier** | **GAP** | competence | ⭐ **belief's missing vocation** — identity verification as a trade |
| **monument mason** | **GAP** | Discipline | ⭐ the **necropolis**; the chronicle made physical |
| ⭐⭐ **knacker · chandler** | **GAP — newly identified 2026-09-03** | premises (industrial) + Discipline | ⭐⭐⭐ **the only genuinely new trade the settlement pass produced.** `organic/leather` ships as a material and so do `hide-stock` / `hide-jerkin` / `leather-boots` / `leather-whip` — **and nothing makes leather**; there is no `tallow`, no `soap`, no `candle`, while Rejection's whole culture section is built on the candle. One input, three outputs: carcass → tallow → **candles (the light need)** + soap; → hide → **the tanner** → leather; → bone → glue. ⚠ Gated on **ranching**. See rendering-slate. |

### Maintain & dispose

| Vocation | Status | Gate | Paid by |
|---|---|---|---|
| roadmender | designed | on-shift capability | the turnpike trust (**wage**) |
| linesman | designed (power-utility) | competence | the utility (**wage**) |
| **water / sewer worker** | **GAP — substrate now shipped** ([watershed.md](./subsystems/watershed.md)) | competence | the utility (**wage**) |
| gatekeeper | designed | employment | the trust (**wage**) |
| **scavenger** | designed (sanitation) | **none — the entry-level vocation** | scrap buyers (**trade**) |
| municipal collector | designed | licence | the locality (**treasury**) |

## ⭐⭐⭐ Teaching — the vocation is not blocked; the CONTENT is

**(User, 2026-07-31, correcting the initial read.)** The gap is **not**
the vocation. It is the **curriculum**:

- **licensed content** is the study.com path;
- **free content** (Wikipedia and similar) is the unlicensed path;
- and **player-authored** courses are the third, which the CMS and the
  authoring tree already support.

### The design

> **NPC instructors, TAs and students populate virtual classrooms — and
> a player may insert themselves at ANY of those roles, provided they
> have the competence the role requires.**

Two things fall out:

- **The classroom runs whether or not players are there** — the same
  DAU-independence commitment as the NPC `hauls` brain. The world does
  not wait.
- ⭐⭐ **Any seat is takeable**, so the competence ladder becomes
  **social**: student → TA → instructor, exactly as real academia works,
  and exactly the *applied hours* thesis.

### ⭐⭐ What is content-agnostic (build now) vs. supply (wait)

**The pedagogy substrate needs no facts in it**: a Course is a document
with a syllabus, the syllabus points at Disciplines, the classroom is a
venue, and the three roles are Positions on a Business. **All of that
ships without a single item of curriculum.**

> **Build the machine now; fill it later.**

### ⭐⭐⭐ And the first curriculum is free, because we built it

The simulation already models — honestly — **the Ohm's-law core**,
**Newton cooling**, **materials response**, **metabolism**,
**respiration**, and **ballistics**.

> **The game's own systems ARE curriculum, and we own them outright.**

So a classroom teaching electrical safety, or materials and how they
respond, needs **zero licensing** — it teaches the world's own physics,
which happens to be real physics. Licensed content extends into
subjects the simulation does *not* model (history, literature, test
prep).

**Which means teaching can ship before any licensing deal.** The
sequencing is: **our own systems → free reference → licensed catalogue
→ player-authored.**

### ⭐⭐ And teaching is categorically unlike every other vocation

For a smith, **competence gates the product**. For a teacher,
**competence *is* the product** — it is the one vocation where the thing
transmitted is the capability itself. Worth remembering when designing
its loop: everything else in this register sells an object or a service;
this one sells what it knows.

## ⭐⭐⭐ The homemaker — the register's second honest NO

**(2026-08-06, from the household pack.)**
The stewardship pillar makes domestic work into real, measured play, which
invites the obvious entry: *homemaker*. It does not survive the test, and
watching **why** is more useful than the verdict.

> **Universal demand meets universal self-service.** Criterion 1 wants
> somebody who *cannot self-serve*; everyone needs their home tended and
> everyone can tend it. Criterion 2 then finishes the job: **if anyone can
> do it, it is a chore, not a job.**

So decompose it — the parts land in five different places:

| Part | Lands as |
|---|---|
| cooking | ✅ **already a vocation** — cook (shipped) |
| cleaning, laundry, tidying **for wages** | ⚠ **texture** — the bottom rung; and it *freezes in absence*, so nobody needs you to do it while they are away |
| reading the condition **band** | 🤖 **automated** — the record already knows it, and it is visible by design |
| ⭐ the **causal** read — which producer binds, and what act clears it | ✅ **a real gate** (below) |
| ⭐⭐ the **continuous** processes of a holding its owner is not in | ✅ **the strongest survivor** (below) |
| child / dependent care | ⛔ **blocked** — no dependents exist (below) |

### ⭐⭐ The register's own test nearly killed the survivor, and that is the interesting part

The obvious rescue is *"the professional isn't better at scrubbing, they know
what needs doing"* — an epistemic gate, which is exactly what the register
predicts (*"if one side knows more, build a vocation"*). But the corollary
bites immediately:

> **Automate what the record already knows; make a vocation of what it does
> not.**

Property condition is a **derived read, deliberately visible** — you and a
prospective landlord can both *see* it. So the diagnosis looks automated, and
the vocation looks dead.

**It survives on a distinction the pillar already draws.** What is visible is
the **band**; what competence buys is the **causal** read — the Stewardship
Discipline's own claim is *"better reads on what a holding needs, wider
maintenance options, the ability to steward more before it slips."* Banding is
presentation, not the mechanism. Knowing you are at `soiled` is free; knowing
the damp is a failed seal rather than your housemates is not.

### ⭐⭐⭐ And the strongest survivor is absence, which the archetypes hand over free

The decisive one falls out of the stewardship
doctrine's decay archetypes rather than from
anything invented for this entry:

> **Room condition FREEZES in absence. Spoilage RUNS over it.**

An absent holder's home does not get dirty — but their **stores rot, their
stock needs feeding, their perishables run their clock**. Somebody has to be
there for the continuous processes while the owner is not, and that somebody
cannot be the owner by definition.

⚠ This matters because it clears the register's hardest brake — **NEVER INVENT
A NEED TO CREATE A MARKET.** No new player obligation is introduced: spoilage's
clock exists for its own reasons, and the demand is *incidental* to it. That is
the innkeeper's honest justification in a different costume (*NPC schedules
make the inn the place that stays open*), and it is why this entry is a **yes**
where "someone should clean your house" is a no.

**So one vocation survives, not a family: the steward.** Gate: the Discipline
(the causal read) **plus capacity**. Ladder: your own home is not a job → a
neighbour's stores while they travel → several holdings → the absent owner's
estate. Failure modes are real and both are the gate's own shadow — misread
the binding constraint, or over-commit capacity and let a client's stores
spoil. *Estate manager* is the top rung of that ladder, never a separate row.

### ⛔ The blocked part, and why it stays blocked

**Nanny / carer fails on supply, not on demand.** The game has no dependents to
care for and that is a deliberate design position, not a gap: a needs-bar child
is the upkeep treadmill pointed at a person, and every clock available to it is
wrong (freeze-in-absence makes the child furniture; run-over-absence makes it a
starvation timer). The pedagogically valuable half of care — teaching someone,
being answerable for their development — is reachable **now** through
mentorship between adults, which the University, the transcript and the
chronicle already support. Keep the row visible and blocked rather than
deleting it; the reason is the useful part.

## Working notes

- ⭐⭐⭐ **The fiscal cycle is the highest-leverage unbuilt system in the
  register** — it is not one gap among four, it is what pays an entire
  column. Every constable, inspector, gaoler, firefighter, clerk and
  public teacher waits on it.
- ~~The largest single civic hole is the fire brigade~~ — **designed
  2026-07-31** (fire § The fire
  service) — which surfaced
  **insurance**, now designed too
  (insurance-slate).
- ⭐⭐⭐⭐⭐ **The information-asymmetry family is now FIVE markets
  deep** — appraiser (condition) · assayer (value) · underwriter (risk)
  · banker (creditworthiness) · credit surveyor (prospects) — which
  yields the register's sharpest test:

  > **If both sides know the same things, build a MENU. If one side
  > knows more, build a VOCATION.**

  That **explains** criterion 2 rather than sitting beside it: **the
  gate is almost always an epistemic advantage.** It is also strong
  evidence the instrumentation build is higher-leverage than its slate
  suggests.
- ⭐⭐ **And its corollary for "isn't this just software?"**:
  **automate what the record already knows; make a vocation of what it
  does not.** The ledger gives payment history free; it does not give
  *prospects*, so somebody has to go and look. (The accountant is the
  hardest case and it holds: **the ledger is cash, accounting is
  accrual** — *when* revenue was earned and *how* an asset is spread are
  judgments the record cannot contain.)
- ⭐⭐⭐ **And its sibling, from the homemaker: universal demand plus
  universal self-service is a CHORE, not a market.** The test generalizes
  past domestic work — it is why barber, cook and launderer are thin
  entries while appraiser and steward are not:

  > **A vocation appears exactly where self-service becomes unavailable
  > (you are ABSENT), uneconomic (you lack CAPACITY), or uninformed (you
  > lack the READ) — never merely because the task is real.**

  A third lens on criterion 2, and the one that catches the failure mode
  the epistemic test misses: a task everybody genuinely needs, that
  everybody can also just *do*.
- ⭐⭐⭐⭐⭐ **The assessed-pays conflict is a CLASS, not a per-industry
  quirk.** Ratings, audits, insurance surveys, building inspections and
  appraisals all share it:

  > **Any assessment paid for by the assessed has ONE conflict and TWO
  > remedies: mandatory publication ("if you commission it, it
  > publishes") and structural separation.**

  State it once; do not rediscover it per industry.
- **Run method 1 against farming and mining next** — they are the two
  chains with builds in flight and unstaffed links.
- **Method 2's current empties**: thermal, concealment, weather, belief,
  chronicle. Each is either a genuine gap or evidence that the subsystem
  serves something other than a livelihood — **both answers are useful.**
- Keep the **gate** column honest. A vocation whose gate is "you decided
  to" fails criterion 2 and will not survive contact with players.
