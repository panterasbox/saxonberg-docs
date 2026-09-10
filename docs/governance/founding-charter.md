# Founding charter (working doc)

> **Status: the founder's self-binding commitment, in force from the first
> dollar.** This charter is the instance-specific instrument the generic
> founding-stake bound requires
> ([draft-constitution.md](./draft-constitution.md) Art. XI). It is **not
> kernel text** — a different operator founding their own instance would
> publish their own charter, or none. It states the **formula** by which the
> founder accrues stake during the fiat phase, and binds from the moment money
> is first taken. It ships with the **stake-ledger slice** (the v1 that turns
> Twitch donations into accrued influence), because the binding *is* the
> ledger: the commitment is enforced by code and published in the
> tamper-evident record, not by promise.

See also:

- [draft-constitution.md](./draft-constitution.md) Art. XI — the generic
  invariant this charter satisfies (formula-fixed · code-enforced · published ·
  auto-sunset → wage-only), and Art. III for what influence is.
- cooperative-slate.md § *Founding vs. amendment —
  the founder's self-binding* — the rationale: why a credible, code-enforced,
  self-diluting commitment is what lets an early backer trust the stake.

---

## Why bind at all

The stake is only trustworthy if the founder cannot quietly rug it. Before
ratification there is **no ratified constitution** to bind the founder — so
the binding cannot be law. It is the same floor that disciplines every
operator in this design: **code, publication, and exit.** The stake ledger
awards *exactly* this formula with no discretionary path to more; every award
is in the verifiable record; a violation is therefore visible, and members may
fork away. The founder binds himself by machine, in public, from the first
dollar — the first real test of the polity's *code-first, integrity-by-
construction* thesis.

## The formula (the fiat-phase stake)

During the founding (fiat) phase, the founder accrues influence by exactly
these terms, and no other:

1. **Producer house — earned, sole-producer, naturally diluting.** The
   founder's producer influence is **earned, not granted**: the game's influence
   system awards it for actual creation (Art. I §3, Art. III) — the same as for
   any producer — the moment that system is online, pre- and post-ratification.
   He sits at ~**100% of the producer house** only because he is *currently the
   only producer*; as CMS authors contribute his share **dilutes by ordinary
   arithmetic**, and he claims no permanent guarantee. Because it is earned, this
   influence is **not a fiat lever** and does **not** sunset (term 4) — the
   founder carries it through ratification as any member carries earned
   influence.
2. **Consumer house — 0%, ceded outright.** The founder takes **no consumer
   influence** — the players' house is theirs from day one. A deliberate
   self-restraint: he earns influence only in the chambers where he bears cost
   (he produces; he matches capital), never in the chamber of participation.
3. **Capital house — matched, plus one.** As money is donated, for every unit
   of capital influence awarded to a donor the founder accrues an **equal
   unit**, **plus one** — a single extra unit that guarantees a working
   majority of the capital house and prevents deadlock, so sweeping early
   changes move without red tape.
4. **Sunset at ratification — of the *granted* levers.** What sunsets is the
   **granted control**: the capital-match and its **+1** margin (term 3). From
   ratification the founder holds **no privileged stake and no guaranteed
   majority**, and accrues only (a) **ordinary influence** for ongoing
   contribution, like any member, and (b) what **legislation grants as wage**
   (Art. XI). His **earned** producer influence (term 1) is ordinary influence
   and carries over — the founder is **not reset to zero**; only the unearned
   control ends.

## What control this is — and how it ends

The founder's grip is **two chambers, eroding to fewer**:

- Producer (~100%, diluting) **+** capital (50% + 1) = **two of three chambers**
  → a majority of chambers → effective control *at the start*, when he bears
  all the risk and does all the work.
- The consumer house is **0%** — entirely the players'.
- As producers join, the producer share dilutes; the founder's control falls
  from two chambers toward one (capital alone, which is not a majority). His
  grip is **strongest now and fades on its own** as the community contributes
  more — the very curve that signals readiness for the **ratifying
  convention.**

So the formula is self-limiting by construction: it grants real control when
the community is just the founder, *withdraws* it as the community grows, and
ends at ratification. But what ends is the **granted control**, not the
founder's **earned place** — the producer influence his work has earned persists
(diluting, legitimate, like any member's), so the founder steps down from
*controller* to *large contributor*, never to zero. The structure — not the
founder's control — is what the polity ultimately entrenches.

## What this charter is not

- **Not a guarantee.** It is a *binding the founder accepts*, not a right the
  founder is owed. Its force is code + publication + exit, not law.
- **Not kernel text.** The generic *shape* (formula-fixed, code-enforced,
  auto-sunsetting) is constitutional; *these numbers* are this instance's, and
  belong here.
- **Not financial.** Stake is not stock (Art. I §1); none of this is ownership,
  dividend, or a redeemable claim. It is governance influence, and it sunsets.

## Buildable now

This charter is **near-term, not parked with the republic**: it ships with the
stake-ledger v1. The ledger that turns donations into capital influence is the
same ledger that accrues the founder's matched-plus-one — one mechanism,
publicly recorded. The sunset is a single condition (ratification) the ledger
honors. Open detail deferred to the ledger build: the exact unit accounting,
the dilution bookkeeping, and the ratification trigger wiring.
