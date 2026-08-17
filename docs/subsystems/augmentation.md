# Augmentation

The substrate for **acquired capability via slotted Stuff** — the
axis distinct from BodyPlan's **innate biological capability**. Mixin
composition expresses capability at build time; augment installation
toggles activation at runtime. A small uniform substrate carries the
union across the codebase.

Wave 1 ships the substrate (`AugmentMixin.confers`,
`MixinApi.getActiveMixins` / `isActive`, the `@RequiresActive`
decorator, the `_augmentGated` / `_grantsModalities` mixin
declarations), the cranial slot on biped/quadruped body plans, the
`AetherImplant` template, and the Avatar bootstrap that
installs it. Wave 2+ adds an install/remove procedure, other
augments, char-gen loadout selection, failure modes / power state,
and the generalized "contribute capability" surface beyond modalities
(verbs, motor, vital functions). See
augmentation-slate.md for the
full roadmap.

## The mental model

- **Mixin composition is build-time semantics** — "this entity HAS
  this capability."
- **Augment installation is runtime semantics** — "the capability is
  currently ACTIVE."

For most mixins (Named, Container, Visible, Mobile, …) the two
collapse: a mixin that is composed is active. For
**augment-gated mixins** (v1: `AetherMixin`) the runtime activation
is conditional on an installed augment conferring it. The substrate
makes the gating transparent — callers see a single
`MixinApi.is<MixinName>` predicate that answers the active question
uniformly across both kinds.

## Substrate

### `AugmentMixin.confers()`

```ts
export interface Augment {
  confers(): readonly string[];  // mixin names this augment activates
}
```

Lives at `lib/augmentation/Augment.ts`. The augment names mixins; the
mixins describe their own grants. Decentralized declarations beat
coupled lists (per the
[antipattern](../antipatterns.md#dont-model-augments-by-listing-grants-directly)).

### Mixin self-declarations

Augment-gated mixins set a static flag plus per-kind grant fields:

```ts
export function AetherMixin<TBase>(Base: TBase) {
  return class AetherMixin extends Base {
    static _mixinName = 'AetherMixin';
    static _augmentGated = true;                            // opt-in
    static _grantsModalities = ['verbal-esp', 'emotive-esp']; // v1 read
    // _grantsLanguages, _grantsAttributeMasks, _grantsVitalFunctions,
    // _grantsSlots — reserved for their consumer subsystems; not
    // read in v1.
    // ...
  };
}
```

The grant shape is **open** — new grant kinds plug in as their
consumer subsystems land. v1 reads only `_grantsModalities` (consumed
by `PerceptionApi.sensorium`).

### `MixinApi.getActiveMixins` / `isActive`

The active-mixin substrate query:

```
active set = {
  m | hasMixin(stuff, m) AND (
    NOT m._augmentGated  OR
    exists augment in slots with m._mixinName in augment.confers()  OR
    m._mixinName in stuff.getSpecies().innateMixins
  )
}
```

The conferral set is the **innate⊕acquired union**: a gated mixin is
active when composed AND (a slot augment confers it **OR** the actor's
species confers it intrinsically — see *Species intrinsic conferral*
below). This mirrors what the sensorium already does for bodyplan senses
and `defaultModeFor` does for the bodyplan locomotion tier.

For un-gated mixins the behavior is identical to today's
`hasMixin`. For gated ones the predicate reflects the
augment-toggled reality — installing or removing an augment takes
effect on the next call (v1 is lazy / uncached; future polish caches
on slot-occupy / release).

The auto-generated `is<MixinName>` predicates (e.g. `isAether`) now
back onto `isActive`. **Uniform calling convention across all
predicates** — no branching on whether a mixin is gated; build-time-
only checks are an edge case using the low-level
`MixinApi.hasMixin(stuff.constructor, name)` directly.

### `@RequiresActive` decorator

Lives at `lib/security/RequiresActive.ts` next to `@CallSecurity` /
`@Final` / `@Unshadowable`. The decorator wraps a method so that at
call time it checks `MixinApi.isActive(this, mixinName)` and throws
`InactiveCapabilityError` when false:

```ts
export class InactiveCapabilityError extends Error { ... }

export function RequiresActive(mixinName: string): MethodDecorator {
  return function (_target, propertyKey, descriptor) {
    const original = descriptor.value;
    descriptor.value = function (this: Stuff, ...args) {
      if (!MixinApi.isActive(this, mixinName)) {
        throw new InactiveCapabilityError(mixinName, String(propertyKey));
      }
      return original.apply(this, args);
    };
    return descriptor;
  };
}
```

**Convention:** every public method on a mixin where
`_augmentGated === true` MUST use this guard. The verb-level
validator (`requiresVerbalESP`, future siblings) is the polite
early-catch at command dispatch time; `@RequiresActive` (or the
equivalent inlined check) is the late-catch for any direct caller
(test fixtures, internal subsystems, NPCs invoking methods
programmatically).

**Implementation note:** TS decorator syntax (TS1206) doesn't apply
inside the *class expression* returned from a class-factory mixin, but
it DOES apply on a *named class declaration* the factory then returns
(`class Foo extends Base { … } return Foo;`) — `AetherMixin` and
`CommandGiverMixin` use that form for their `@Final @Unshadowable`
chokepoints. The transmission guard moved to `CommsMixin.tell` (the
hosted comms update), which checks its operator (`getHost()`) is an
aether host. The decorator stays the canonical form for non-mixin call
sites.

## The cranial slot

Biped and quadruped body plans declare a `cranial` slot of capacity
1 that accepts `SlottableMixin`:

```yaml
# seeds/lib/body-plans/biped.yaml
slots:
  # ...
  - { name: cranial, accepts: SlottableMixin }
```

The "implant" kind is documented prose on the slot's name, not a
new validator enum. Sessile body plans deliberately omit the
cranial slot (plants don't get implants in v1).

## `AetherImplant`

The Wave 1 implant template:

```ts
// lib/augmentation/AetherImplant.ts
export class AetherImplant
  extends AugmentMixin(SlottableMixin(TangibleMixin(Thing)))
{
  static readonly TEMPLATE_PATH = '/obj/AetherImplant';

  override confers(): readonly string[] {
    return ['AetherMixin'];
  }
}
```

Hardened per the slate: no power state, no failure modes, no fuel.
Diegetically a small brass-and-silicon device. The implant is purely
the **attunement conferrer** now — it no longer carries any credential
directly (those live in the hosted `CredentialWalletUpdate`).

`Avatar.installDefaultLoadout` (dispatched from `postRegister` during
the clone cascade, once per session) keys off **whether the avatar is
attuned by any source**: if `AetherMixin` isn't already active (a
born-attuned species confers it intrinsically — see below), it occupies
the implant in the cranial slot to confer it; otherwise it skips the
implant. Either way the avatar is now an `AetherHost`, so the loadout
injects the default updates (comms, forums, and the credential wallet)
onto it. Idempotency keys off **"already hosts a comms update"** — correct
for both the implanted and born-attuned paths.

## How other augment kinds plug in (substrate proof)

| Augment Stuff | `confers()` returns | Mixin's grants | Consumer subsystem |
|---|---|---|---|
| `AetherImplant` (v1) | `['AetherMixin']` | `_grantsModalities` | `PerceptionApi.sensorium` |
| `ThermalVisionImplant` (future) | `['ThermalVisionMixin']` | `_grantsModalities` | `PerceptionApi.sensorium` |
| `CyberArm` (future) | `['ProstheticArmMixin']` | `_grantsSlots`, `_grantsAttributeMasks` | `SlotApi`, `PropertiedMixin` |
| `TranslationChip` (future) | `['TranslationMixin']` | `_grantsLanguages` | `LanguageApi` (future) |
| `ArtificialHeart` (future) | `['HeartFunctionMixin']` | `_grantsVitalFunctions` | `VitalsApi` (future) |

v1 ships only the first row. The rest demonstrate that the same
substrate handles each augment kind — new augments need no
framework change, only:
1. a new Thing template composing `AugmentMixin` whose `confers`
   names the relevant mixin, and
2. that mixin declaring its grants.

## The three-base capability model + the aether hosting relation

> **Canonical statement.** A **capability is a mixin bundle
> manifestable around three bases** — a corporeal **`Thing`** (carried),
> an incorporeal **`Idea`** (an *update* hosted on aether attunement),
> or **intrinsically** on a Creature / species — and **one reachability
> scan (the MQL `reachable` seed) finds it in any form.**

`AetherMixin` is no longer a comms-carrying mixin: it is the aether
**host**. *Attunement* is the conferred (implant) or intrinsic (species)
capability whose payload is a host that aether `Idea`s (updates) plug
into. Comms and the credential wallet are hosted updates; the physical
`TravelCard` / `PaymentCard` (and a future radio) are their corporeal twins.

### The hosting relation (distinct from containment)

- **Update side** — `AetherHostedMixin` (`lib/augmentation/AetherHosted.ts`)
  composes around an `Idea`: a `getHost()`/`setHost()` back-ref and the
  must-be-hosted invariant. The capability mixin (`CommsMixin`,
  `CredentialWalletMixin`) co-composes on the same update class
  (`CommsUpdate`, `CredentialWalletUpdate`).
- **Host side** — `AetherMixin` carries `_hostedUpdates` (the
  source-of-truth collection) and the sealed chokepoints
  `hostUpdate` / `unhostUpdate` (`@Final @Unshadowable`).
  `hostUpdate` pushes + back-refs + notifies command routing;
  `unhostUpdate` removes + retires contributions + **destroys an
  orphaned update** (the must-be-hosted invariant).
- **The invariant lives on the relation, NOT on `Idea`.** A bare `Idea`
  (Biome / Zone / Controller) composes no `AetherHostedMixin` and is
  unaffected — it stays free-standing-capable. There is **no hosting
  Api**: the relation is orchestrated by the host mixin methods plus
  the MQL `reachable` seed for lookups (a narrow `IdeaHost` mixin would
  graduate only if a *second* incorporeal-host concept appears).
- **Lifecycle** — a hosted update is host-bound: `AetherMixin`'s
  `cleanupOnDestruct` destroys every hosted update when the host dies
  (Avatar logout), mirroring how the implant clone died with the avatar.

### Reachability: the self + host-descent legs

The MQL `reachable` seed (`api/mql/scope-walk.ts`
`candidatesForReachable` — it absorbed the walk of the retired
`ContainmentApi.findReachable`) carries two legs on top of the slot →
inventory → location walk, preserving on-your-person-first order:

1. **Self leg** — `predicate(actor)` (a capability composed directly on
   the actor / its species).
2. **Descend-into-host leg** — for the actor itself and each carried /
   slotted attuned host, also test its hosted updates (the implant-hosted
   comms / credential on self; the future carried radio's comms).

One pool finds the capability whether it is a carried `Thing`, a hosted
update, or composed intrinsically: consumers resolve
`MqlApi.resolveMany('reachable', …)` and narrow locally with
`MixinApi.isX` — `.find` over the on-person-first pool keeps the old
first-match contract. The host-descent leg is bounded to a single
concept and a single level; do not teach the seed another leg.

### The contribution walks generalize (no parallel path)

The surfaces that already aggregate slotted-augment contributions extend
to include a host's hosted updates:

- **Command-source** — `CommandApi.collectHostedUpdateDefs` +
  `applyHostedUpdateDelta` push each hosted update's `self`-bucket verbs
  onto the host's recency stack with the update as `commandSource`.
  `CommandGiverMixin` seeds them in `postRegister` and the lazy
  `_ensureSelfEntry` safety net; host/unhost surfaces/retires them live
  (gain-/lose-post-spawn).
- **Sensorium** — `walkAugmentedModalities` unions `_grantsModalities`
  from the host's hosted updates' active mixins. Substrate-only in v1
  (no update grants a modality; attunement does).

### Species intrinsic conferral (the innate leg)

`Species` declares `innateMixins: string[]` — authored data, the mirror
of `AugmentMixin.confers()`. `collectAugmentConferralNames` unions it
with slot-augment conferral, so a born-attuned species activates
`AetherMixin` with no implant. **Scope**: this *activates a gated mixin
already composed on the shared `Creature`/`Avatar` class* — it cannot
compose a new mixin onto an instance (deferred). A throwaway test
species (`homo/sensitivus`) exercises the path.

## Wave 1 boundary

What ships:
- `AugmentMixin.confers()` + `_augmentGated` / `_grantsModalities`
  mixin declarations.
- `MixinApi.getActiveMixins` / `isActive` + predicate routing.
- `@RequiresActive` decorator (inlined equivalent in AetherMixin).
- Cranial slot on biped/quadruped body plans.
- AetherImplant template + Avatar.installDefaultLoadout (from
  postRegister) clone-time install.
- `requiresVerbalESP` + `requiresEmotiveESP` verb-level validators.
- Reception-gating integration (`Scene.modality` + filterMessage)
  drops `dm` frames for implant-less recipients.

What's reserved for Wave 2+ (see slate):
- Install / remove medical procedure (vitals tie).
- Other augments (translation, prosthetics, sensor packages, motor /
  cognitive).
- Char-gen loadout selection.
- Failure modes / power state / jamming / spoofing.
- Magic-flavor augments.
- The generalized "contribute capability" surface beyond modality
  grants (verbs, motor, vital functions).
- ~~Methods / state CONFERRED by augments — today they live on the
  natively composed mixin, with the augment acting as the activation
  flag. Future refactor moves AetherMixin off Avatar and onto the
  implant Stuff itself (verb dispatch routes through the augment).~~
  **Realized (capability-hosting build):** the comms capability moved
  off `AetherMixin` onto a hosted `CommsUpdate`; verb dispatch routes
  through the update as `commandSource`. See *The three-base capability
  model* above. `AetherMixin` is now the host, not the carrier.

## Conventions

- Augment-gated mixins MUST guard every public method
  (`@RequiresActive('<MixinName>')` or inlined equivalent).
- Don't mint a parallel `isFooActive` predicate when adding an
  augment-gated mixin. `MixinApi.isFoo` already answers the active
  question uniformly.
- Build-time-only composition checks (rare) use
  `MixinApi.hasMixin(stuff.constructor, name)` directly.
- Don't model augments by listing their grants directly. An augment
  declares which mixins it confers; the mixin declares what each
  grant kind contributes.

## File layout

```
lib/augmentation/
├── Augment.ts                  AugmentMixin
├── AetherHosted.ts             AetherHostedMixin (update side of hosting)
├── AetherImplant.ts      Wave 1 implant template (attunement only)
└── __tests__/Augment.test.ts, AetherHosted.test.ts

lib/comms/
├── Comms.ts                    CommsMixin (transmission, hosted update)
└── CommsUpdate.ts              CommsMixin(AetherHostedMixin(Idea))

lib/message/Aether.ts           AetherMixin = attunement + host side

lib/security/
├── RequiresActive.ts           Decorator + InactiveCapabilityError
└── __tests__/RequiresActive.test.ts

seeds/lib/augmentation/
└── AetherImplant.yaml    Hydrator-ready seed

api/mixin.ts                    getActiveMixins / isActive
                                + isAugment predicate
lib/mixin.ts                    Mixins.Augment registry constant
```

## Cross-references

- augmentation-slate.md — the
  slate this build's Wave 1 graduates.
- [senses.md](./senses.md) — the perception substrate that consumes
  `_grantsModalities` via `PerceptionApi.sensorium`.
- [race.md](./race.md) — the cranial slot addition to biped /
  quadruped body plans.
- [slot.md](./slot.md) — the unified slot universe the cranial slot
  uses; no parallel `implantSlots` field.
- [messaging.md](./messaging.md) — gated reception via per-frame
  `meta.modality`; transmission now rides the hosted `CommsUpdate`.
- [comms.md](./comms.md) — comms-as-update (`CommsMixin` on
  `CommsUpdate`), `tell` sends from `getHost()`.
- [fasttravel.md](./fasttravel.md) — the `travel` credential, now a record
  in the unified `CredentialWalletUpdate` + the `TravelCard` Thing twin.
- [credential.md](./credential.md) — the credential-wallet substrate that
  unified the payment + travel hosted updates into one wallet app.
- [perception.md](./perception.md) — viewer-aware-query pattern;
  augment activations flow into the sensorium walk transparently.
- [antipatterns.md](../antipatterns.md) — augment-modeling
  antipatterns this substrate forbids.

## History

Wave 1 (substrate + `AetherImplant`, 2026-06) shipped `AetherMixin` as
a comms-*carrying* mixin composed natively on `Avatar`, gated active by
the implant. The **capability-hosting build** (2026-06) then realized
the reserved Wave-2 item — *"move AetherMixin off Avatar; verb dispatch
routes through the augment"* — by introducing the **three-base
capability model**: `AetherMixin` demoted to the aether *host* (host
collection + ESP modalities), comms transmission moved out to a
`CommsMixin` hosted on a `CommsUpdate` `Idea`, the travel credential
became a `TravelCredentialUpdate` (the implant stopped carrying it), and
`Species.innateMixins` added the intrinsic conferral leg. The plan
proposed a possible `AetherHostApi`; implementation rejected it — the
relation is orchestrated entirely by the `AetherMixin` host methods plus
the reachable scan (today MQL's `reachable` seed), per the no-new-Apis
rule. See the
`feat(augmentation): capability hosting` commit and its review-cleanup
follow-up.

The **credential-wallet build** (`feature/credential-wallet-build`) later
collapsed the per-credential hosted updates (`PaymentImplantUpdate` +
`TravelCredentialUpdate`) into a single born-with `CredentialWalletUpdate`
holding credentials-as-data — the loadout now injects one credential app,
not one per kind. See [credential.md](./credential.md).
