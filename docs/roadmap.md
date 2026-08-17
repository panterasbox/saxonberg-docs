# Saxonberg 2.0 Roadmap

Forward-looking work, organized by area. The phase numbering from
the legacy `PLAN.md` is no longer load-bearing; treat it as
historical.

This roadmap is a navigation aid. Concrete design lives in slates
under `docs/`; implementation guidance lives in `docs/architecture.md`
and the subsystem references under `docs/subsystems/`. The
design-philosophy.md is the principle
that shapes every slate.

> **Sequenced sibling (2026-07-31):**
> living-world-roadmap.md — a
> **dependency-ordered** nine-phase build sequence for the family of systems
> that *grow, are kept, sicken, spoil and are worked* (farming · ranching ·
> pets · preservation · disease · the health vertical · stewardship · seasonal
> labor). This doc organizes **by area**; that one organizes **by order**, and
> is narrower and newer where they overlap. **Phases 1 and 2 are shipped:**
> the houseplant / growth model
> ([husbandry.md](./subsystems/husbandry.md)) and Hinkley Hills — ground
> you own, the garden bed, the harvest and land use
> ([smallholding.md](./subsystems/smallholding.md)).

---

## Foundation (shipped)

The substrate is in place. Major shipped surfaces:

- **Auth + persistence** — Google OAuth; the **self-persistence spine**
  (`PersistableMixin`/`PersistableApi` — any host captures its own fields +
  inventory + gear + location into `holder_snapshots`, routed through
  call-security; **multi-instance nested hosts** ride a keyed `{ref, key}`
  reference, so many instances of one template nest without collapsing;
  Avatar migrated onto it, so gear/inventory now survive
  logout); template/clone track for the Idea hierarchy; Marshaller
  framework for non-default serialization.
- **Standard Model + mixins** — class-factory mixin pattern,
  `Mixins` registry, composition rules; `PropertiedMixin` for
  typed properties.
- **Spatial substrate** — `Stuff` / `Idea` / `Thing` /
  `Location` / `Vessel`, `Cartesian` / `Spherical` zones, exits,
  doors, windows, the Boundary substrate (Adornable +
  Adornment + Conduit), Sealable.
- **Light & Boundary subsystem** — Light value object,
  propagation walk via `VisionModality.signalAt` (called directly on the
  vision singleton), per-viewer perception
  (`VisionModality.canSee`, `perceivedBand`), the Boundary
  substrate's channel-keyed transmissivity (one of five conduit
  kinds shipped with the perception substrate).
- **Quantities substrate** — `Quantity<U>` value object,
  per-unit math op table, tag-table registry, YAML-authored
  scales, `<quantity>` Mml emission (registered, non-inert, and
  carrying `channel` / `via` / `lo` / `hi` so a reading survives the
  wire as a number with provenance), marshaller integration.
  Consumed by Light (lux/lumen/Kelvin), Material (kg/m³, g/mol),
  Tangible (kg); future channels (sound, heat) plug in via the
  same shape. See [docs/subsystems/quantities.md](./subsystems/quantities.md).
- **Race / species / organism (v1)** — Material substrate, Clade
  taxonomy, BodyPlan + Species, OrganismMixin, SexedMixin,
  SpeciesApi (kingdoms, lifecycle predicates, `isAnimate`).
  Animacy gating on commands. v1 acceptance roster: Homo
  sapiens, Homo khazadicus, Lithobates catesbeianus,
  Spathiphyllum wallisii, Constructa metallica.
- **Command framework** — YAML view + controller MVC, validators
  (field + verb-level), MQL grammar (pronouns, multi-select,
  chains, filters, focus), per-character aliases (`AliasMixin`).
- **Shell** — `EnvironmentMixin` (settings keyspace),
  `WorkspaceMixin` (`pwd`/`cd`/`ls`/`cat`/`grep`/`write`/
  `mkdir`/`rm`/`cp`/`mv`), `AuthorMixin` (`clone`/`reload`/
  `destruct`/`eval`/`teleport`), `PerceiverMixin` (`look` /
  `scry` / `locate`), prose / liquid templating.
- **Communications** — `Sensor` / `Vocal` / `Mobile` mixins;
  `say` / `tell` controllers; messaging subsystem (MML, scene
  composer, movement-message settings); `MudlogApi`.
- **Event system** — `EventApi` global pub/sub bus, Witness
  pattern, `EventRegistry` Idea bootstrapped via
  `BootstrapManager`. Spatial-mixin lifecycle hooks dispatched
  through `ContainmentApi.move`, `Mobile.traverse`,
  `ConnectionApi`.
- **Module hot-reload** — `HotReloadApi` with module registry,
  dependency graph, admin `reload` verb,
  `StuffApi.clone`-integrated lifecycle.
- **Call security** — proxy interceptor pipeline, decorators,
  policies, `@CallSecurity` / `@Final` / `@Unshadowable`,
  `SecurityApi.decorateApiClass`.
- **Bootstrap + state model** — `BootstrapManager`, unified
  state model (Shadow, PostRegistration), HomeZone for
  per-player namespace at `/home/<playerId>`.
- **Spawn shape (declarative authoring)** — Template
  `environment:` field, `PopulatesMixin`, escape hatch via
  `PostRegistrationMixin`. Shipped — see
  [subsystems/templates.md](./subsystems/templates.md),
  [spatial.md](./subsystems/spatial.md), and
  [boundary.md](./subsystems/boundary.md) (deferred notes preserved in
  templates.md).
- **Client/server wire substrate** — the cockpit's foundation,
  shipped:
  [response-envelope.md](./subsystems/response-envelope.md)
  (structured machine channel beside MML),
  [events.md](./subsystems/events.md) (class-per-event
  vocabulary on `EventApi`),
  [mql-subscription.md](./subsystems/mql-subscription.md)
  (client-driven live-state: subscribe via MQL, server diffs
  deltas), [prompt.md](./subsystems/prompt.md) (`PromptApi`
  Tier-1 surface + cardinality/onExcess disambiguation), and
  [card-surface.md](./subsystems/card-surface.md) (the right
  column as a feed of cards — one birth path, pinned as the whole
  lifetime, liveness scoped to attention, one inspection card laid
  out by `StuffKind`). These graduated from slates to subsystems.

See [docs/architecture.md](./architecture.md) for layout and
docs/subsystems/ for individual references.

---

## What's next — two parallel tracks

> **Near-term priority now lives in
> launch-worklist.md** — the guild-derived
> economy gap list (2026-07-28): the extraction faucets, the transform
> chain, the polity-paper demand floor, and the sinks that must exist
> before a persistent beta, readiness-tagged. The track structure
> below remains the substrate map.

The substrate is deep enough that work now splits cleanly into two
tracks that run **in parallel**, distinguished by whether a client
dependency gates the work:

- **Track A — Client (cockpit).** Rides the shipped wire
  (mql-subscription + prompt + envelope). Each item is a *tandem
  slice* (smallest server addition + matching widget) or
  *client-pull* (server already has what it needs; the client
  iterates). The inspection pane already landed this way.
- **Track B — Server substrate.** *Server-first*: pure mechanism
  that builds and proves out against the existing text client and
  the shipped wire — **no client blocker**. Verbs like `analyze` /
  `say` / `wear` exercise it without any cockpit work.

The taxonomy (server-first / tandem / client-pull) and the
prioritized near-term work live in this section. The full
slate catalogue (the menu these are drawn from) follows under
**Active design slates**.

### Track A — Client (cockpit), near-term

The two original near-term items both shipped (message-rendering
2026-06; prompt-stack absorbed into the CommandBar's slot multiplexer
during the console-foundations merge — see retired list below). What
remains in the near-term Track A queue:

1. **Client-pull cockpit buildout** — the layout/composition substrate
   shipped (cockpit-layouts build, 2026-06: server-authoritative `layout`
   axis + per-bar input mode + livestream-viewer + builder + the
   summoned-pane tier — see
   [cockpit.md](./subsystems/cockpit.md)). What remains:
   the education content surface (video + transcript), the study /
   classroom / tutor modes, and theming polish. Per
   client-cockpit-slate.md;
   decomposes into per-track requirements docs at build time.
2. **Scoped-authoring GUI** (the room editor) — blocked on
   access landing in Track B; will pair with the new-player flow.
   (scoped-authoring-slate.md)

> **Retired from Track A** (so future passes don't re-suggest):
>
> - **Message-rendering.** Shipped 2026-06 — see
>   [message-rendering.md](./subsystems/message-rendering.md).
>   Nested-aware renderer, stylesheet engine + three themes
>   (`ink`/`marble`/`high-contrast` — the civic ground, 2026-08),
>   per-message-type templates, Discord-dialect markdown, custom URI
>   schemes, mentions, `style` verb + overlay.
> - **Prompt-stack client UI.** Shipped in the console-foundations
>   merge. The CommandBar is a slot-multiplexed input that absorbs
>   every Tier 1 kind (choice / confirm / text / mql-object /
>   mql-many) directly — no separate `PromptArea` component. Slot
>   picker, draft state per slot, chip affordances, X-cancel, mode
>   sigils + tints, echo-snapshot pairing all land there.
>   (prompt-stack-slate.md stays
>   open for Tier 2/3 kinds and future-wave server polish.)
> - **Vitals tandem slice.** The original "HP/MV widget over
>   `me.vitals`" framing reflected the scalar-HP shape Vitals had
>   before the slate was rewritten. Vitals is a full server substrate,
>   not a client first-vertical. The **substrate (models) has now
>   shipped** — the `Creature` body/agency split, vital signs,
>   `vitalProfile`, typed anatomy + tissue, the two-kind condition type
>   system, the generalized `Reserve` substrate, and the death/
>   consciousness seams ([vitals.md](./subsystems/vitals.md),
>   [reserve.md](./subsystems/reserve.md)). The first *drivers* on the
>   substrate have since shipped: **encumbrance** (the carry-weight gauge,
>   [encumbrance.md](./subsystems/encumbrance.md)) and **metabolism** (the
>   intake-and-chemistry driver — `eat`/`vomit`, the digestion buffer,
>   coupled recovery, and the first condition-*driver*: floored reserves
>   spawn `starvation`/`dehydration`/`collapse` and progress to the death
>   seam, [metabolism.md](./subsystems/metabolism.md)), and **respiration**
>   (the air-exchange driver that finally drives `spo2` to an anoxia death
>   seam on drowning / vacuum, plus the carried-air scuba loop,
>   [respiration.md](./subsystems/respiration.md)), and **harm** (the
>   first *injury* driver — the `inflict` producer, live trauma
>   progression reconcile-on-read, bleed → `exsanguination`, and the
>   non-combat medic vertical `assess`/`treat`/`dress`/`undress` +
>   the `DressingMixin`/`Bandage` capability,
>   [harm.md](./subsystems/harm.md)). Still deferred: the *general* /
>   instrument-mediated assess + measure-on-patient, the first-aid
>   instrument + medicine branches (splint / suture / antiseptic — harm
>   shipped only the dressing branch). The *general* death-transition
>   driver SHIPPED 2026-08-01 — every driver now routes through one
>   `ConditionApi.die` ([mortality.md](./subsystems/mortality.md)).
>   (vitals-slate.md)
> - **Things-here / Inventory chip strips.** Drafted as
>   tandem-slice training wheels before the inspection pane
>   shipped. The pane's `here` + `contents` payload already
>   renders clickable room contents inline; ambient awareness of
>   state changes rides the terminal scrollback. No compelling
>   user need surfaced in 2026-06-06 scoping. The cockpit slate's
>   panel inventory (Room-state row) already reflects this
>   absorption.
> - **Console filtering drawer.** **Shipped** in the
>   console-foundations merge (gutter stripe + filter drawer +
>   mute-count badges + topics + tabbed terminal +
>   `ClientStateMixin` substrate).
>   (console-filtering-slate.md)

### Track B — Server substrate, near-term

Ordered by leverage + dependency:

1. **Access / capability `can()`** — foundational; gates
   scoped-authoring and spoiler, and absorbs the
   communication-policy slate. Pure server. **Wave 1 shipped
   (2026-06)**: `AccessApi` thin facade over the singleton
   `AccessRegistry` Stuff with four predicates
   (`can` resource-targeted slice walk, `canMutateZone` role-gated,
   `isAuthor` broad content-scope, `isWizard` orthogonal
   TS-escape axis — renamed from `isDeveloper` by the wizard-authority
   build) + `resolveSourceFolderZone` path resolver;
   `Zone.ownerGroup` / `accessGroups` persistent inheritable fields;
   the narrow-entry pattern (`FromController` policy applied to
   `StuffApi.forceDestruct` + `ContainmentApi.forceMove`); the
   three bootstrap-seeded groups (`'core'` / `'lounge'` /
   `'wizards'`, renamed from `'developers'`) with the lounge FolderZones; MQL
   `ctx.permission` snapshot wire-up; access checks on all 13
   author-tier and workspace verb controllers; two declarative
   validators (`requiresCoreAccess`, `requiresWizard`). See
   [access.md](./subsystems/access.md). Wave 2+ deferred:
   possession / ownership / location capability sources,
   action-level enforcement for chat / door / channel,
   deny-wins composition, audit sink wiring (call-security
   Pillar 5), tier vocabulary, class-allowlist for content
   writes.
   (access-slate.md)
2. **Senses** — the unified `Modality` substrate (sound = the
   hearing channel); high content leverage, `analyze`/`measure`
   are text verbs. **Wave 1 shipped** (2026-06): authoring surface
   (per-sense `Detail` slot map, `<sense channel="X">` MML,
   `senseStripAugmenter`, four single-sense verbs, gestalt `sense`
   verb, auto-on-entry, hierarchical perception topic tree) AND
   the physics substrate (`Modality` base + seven singletons +
   `PerceptionApi`; field propagation walks for vision / smell /
   sound; touch ambient + per-detail temperature via biome chain;
   ESP via augment-conferred AetherMixin; per-frame modality
   attribution + reception gating; ambient producers via
   `Biome._defaultAmbientSoundLevel`). See
   [senses.md](./subsystems/senses.md). Still ahead (Wave 2/3):
   smell trails / temporal persistence, active-sense pattern
   (echolocation), full ESP local-field walk, per-species
   hearing / tactile / gustatory profiles, RT60 / reverberation,
   stealth as sensorium-relative perception.
   (senses-slate.md)
3. **Social cluster (server halves)** — **shipped (2026-06)**:
   `SoulMixin` emotes with `EmoteGrammar` slot binder + ~35-emote
   starter roster + `:` / `;` prefix dispatch; comms acoustic
   cleanup (whisper / shout / `say --to`) with `meta.acousticDb`
   stamps; `GroupApi` facade over four providers (managed / MQL /
   contacts / channel); `ContactsMixin` personal lists on Avatar;
   chat substrate with `Channel` Documents, `ChannelCatalogue`
   singleton, and the Phase 1 opt-in `fallthrough: true` command
   framework flag; DM hardening (multi-target via collapsed
   `tell` surface, reply/reply-all, cohort state — later moved to
   the hosted comms update by the capability-hosting build);
   `broadcast` verb; **act-scoped emote reactions** (the
   aggregate-delta substrate + `react` verb + chip UI — see
   [reactions.md](./subsystems/reactions.md)). See
   [emotes.md](./subsystems/emotes.md),
   [grouping.md](./subsystems/grouping.md),
   [chat.md](./subsystems/chat.md),
   [contacts.md](./subsystems/contacts.md). Still ahead: Layer 3
   entitlements; chat Wave 2+ (role overlay,
   config block, edit/delete, mentions, pinned, directory);
   broadcast permission gate (waits on access slate); rate
   limiting; vitals-aware acoustic dB; recognition-family
   buildup over the contacts substrate.
   (emotes /
   comms /
   chat /
   social-graph)
4. **Collision** — *decomposed; not a standalone near-term pick
   (resolved 2026-06-10).* The slate bundled three different-priority
   concerns that disperse rather than ship as a unit:
   - **Intentional blocking** (the gate guard) is **behavior**, not a
     mixin — it lands as a **`guards` brain** under
     npc-behavior, riding a small
     block-substrate seam in the locomotion cascade (the
     already-present-but-hollow `'blocked'` gate in
     `LocomotionControllerBase.composeRejection`). The slate's
     `BlockerBehavior`-as-composed-mixin is the *wrong shape* under the
     brain model (behavior = data configuring a swappable brain, never a
     bespoke mixin). The diegetic "why you can't pass" requirement is
     already ~80% met by the engine's per-gate, source-attributed
     rejection prose (doors say *"The oak door is closed."*).
   - **Capacity** (room/vessel occupancy caps) and **pushing**
     (`Pushable` + `PushActivity`) are **defer-til-content** — a field +
     validator, and a verb + activity respectively, each cheap to add the
     day specific content asks.
   So don't re-surface collision as "what's next": its live half is a
   brain owned by npc-behavior; the rest waits on content.
   (collision-slate.md)
5. **Recognition family** — recognition (full) + identification
   (substrate) + the viewer-aware naming step (the realized
   `DescribeApi v2`) **shipped 2026-06** as the belief subsystem; see
   [belief.md](./subsystems/belief.md). The **`regard` facet** (the third
   belief realm — per-viewer signed attitude, `RegardApi`/`RegardLogic`,
   the reverse `{realm,referent}` index) then **shipped 2026-06** as the
   first brick toward reputation / the Compact's Sybil keystone.
   The **renown aggregation substrate** then **shipped 2026-06**
   ([renown.md](./subsystems/renown.md): per-scope signed standing, decay,
   the reaction + reception signal generators, log-saturation). The
   **influence stocks** then **shipped 2026-06** as its first consumer
   ([influence.md](./subsystems/influence.md)): the consumer (engagement
   meter), **producer** (make) faucet, the **conviction** spend substrate
   (abstain/quorum, no verb yet), the durable-`templatePath` re-key, and the
   **authorship ledger** ([provenance.md](./subsystems/provenance.md)) — the
   capital faucet, the ballot/chambers, and NPC-susceptibility / eigenvector
   weighting stay deferred. The **social-graph attention layer**
   (crowd-bucket verbosity + the `notify` notification policy) then
   **shipped 2026-06** as social-graph Wave 3
   ([social-graph.md](./subsystems/social-graph.md)); its remaining tail
   is Wave 4 (account-level federation) + the message-restyle live wiring.
   Remaining tails: identification's
   **pedagogical instrument seam** (`analyze X with Y`, real Material
   chemistry) + partial-ID + misidentification, and the **aether
   id-aug** ambient trigger (axes unresolved + aug substrate retooling).
   (reputation /
   social-graph /
   identification)
6. **Augmentation Wave 2+ + npc-dialogue** — Wave 1 augmentation
   shipped 2026-06 (substrate + AetherImplant; see
   [augmentation.md](./subsystems/augmentation.md)); the
   capability-hosting build (2026-06) then shipped the **three-base
   capability model** + aether-as-host (comms + travel credential as
   hosted updates, the reachable scan's self + host-descent legs (now
   the MQL `reachable` seed), species
   intrinsic conferral). Wave 2+ adds the install/remove medical
   procedure, char-gen loadout, other augment Stuff (translation,
   prosthetics, sensor packages, motor / cognitive), and failure
   modes. Npc-dialogue **Wave 1 shipped 2026-06** (the branching-tree
   responder + `talk to` + auto-introduce; see
   [npc-dialogue.md](./subsystems/npc-dialogue.md)); the scripted
   `intent-dialogue` / LLM front-end is the tail.
   (augmentation /
   npc-dialogue)

The **new-player flow** straddles both tracks: char-gen engine +
fast-travel terminals are Track B (server-first) — the **char-gen
engine shipped in Wave 1** (see `docs/subsystems/char-gen.md`); the onboarding
journey content + the authoring GUI are Track A. Reactions, chat,
and spoiler each have a server half (Track B) and a client surface
(Track A) — land the server half first, surface it when the
cockpit reaches it.

---

## Active design slates

The exploratory design pass — the **catalogue** the two tracks
above draw from. Each slate is a working doc shaped for review;
concrete implementation follows when a slate is promoted to formal
requirements.

### Top-level guidance

- docs/design-philosophy.md — "model
  the smallest fidelity content needs, do it honestly, present
  in layers." Spatial-fidelity axis; ranged-action and capacity
  worked examples.
- docs/runtime-model.md — Node event
  loop, timing primitives, wire transmission, multi-client
  reality, isolation tradeoffs. Reference doc consumed by
  slates that schedule work.

### Substrate slates

- **Embodiment subsystem (shipped)** — see
  [docs/subsystems/slot.md](./subsystems/slot.md),
  [embodiment.md](./subsystems/embodiment.md),
  [posture.md](./subsystems/posture.md),
  [conveyance.md](./subsystems/conveyance.md). Slot substrate
  (`Slotted` / `Slottable`); body-side affordances
  (`Wearable`, `Wieldable`); world-side (`Postured`,
  `Mountable`, `Drivable`). Conveyance ripple via
  `Mobile.traverse`. Floor adornments and the
  ground-targeting path for sit/lie/kneel.
- **Locomotion subsystem (shipped)** — see
  [docs/subsystems/locomotion.md](./subsystems/locomotion.md).
  `LocomotionMode` singletons; verb-as-mode dispatch
  (`walk`/`climb`/`swim`/`fly`/`ride`/`drive`); enablement
  mixins (`Climbable`/`Swimmable`/`Flyable`); `LocomotionApi`
  emission walk; four-gate cascade (body-plan, posture,
  exit.canTraverse, enablement). The original locomotion
  slate was retired. Its forward-looking **trap / detection /
  run-as-mode** notes have now **shipped** — the concealment /
  detection gate + the `HazardMixin` trap substrate + the
  `sneak`/`run` care↔speed modes (see
  [concealment.md](./subsystems/concealment.md) /
  [hazard.md](./subsystems/hazard.md)); **pathfinder** remains
  the open forward note.
- [docs/subsystems/activity.md](./subsystems/activity.md) — Wave 1
  substrate shipped: `SchedulerApi`, `EngagedMixin` on `Character`,
  engagement slots, the five framework-intrinsic abort reasons,
  HMR-aware lifecycle dispatch, `cancel` verb. Locomotion-as-
  activity and host-slot activities deferred — see
  docs/slates/tails/locomotion-as-activity-slate.md
  and
  docs/slates/tails/host-slot-activities-slate.md
  for the design sketches.
- **Time subsystem (shipped 2026-06)** — see
  [docs/subsystems/time.md](./subsystems/time.md). `WorldClockApi`
  own-thing game-time axis (scale/pause/resume, `WorldClockState`
  persistence, crash backstop, `SystemRoot`-gated lifecycle); the
  arm-next-deadline heartbeat driving `after`/`at`/`every`/`onDate`/
  `cron`; `SchedulerApi` moved onto game-time (D5). `CelestialApi`
  real solar/lunar geometry + `CelestialProfile`/`EARTH_LIKE`
  (compute only — **no celestial→light wiring**, deferred until the
  perception branch merges). `DefaultCalendar`. Pedagogical surface
  (`Sundial`/`Sextant`, `analyze time`/`sky`, `measure shadow`/
  `altitude`). [time.md § Future work](subsystems/time.md#future-work)
  holds the deferred surface (light wiring, multi-region latitude,
  locale, weather, NPC schedules, second profiles).
- **Sound** — **absorbed into
  senses-slate** (now the *hearing* instance of
  the unified `PerceptionChannel` substrate). The standalone sound-slate
  was retired; its acoustic detail (real dB / Hz / species hearing ranges,
  acoustic instruments) is folded into senses-slate's "Deep acoustic spec"
  section — the live authority.
- docs/slates/tails/augmentation-slate.md —
  the augmentation umbrella (implant / prosthetic / graft; innate ⊕
  acquired): a **slotted Stuff contributes a capability** (sense channel,
  verb, comm transport); the baseline comm implant; cybernetic flavor,
  flavor-agnostic substrate. Surfaced by comms (the ESP transport) and
  char-gen (issued at intake).
- docs/slates/deferred-rpg/collision-slate.md — capacity
  (typed-list-of-constraints), intentional blocking, pushing (`Pushable`
  + `PushActivity`). **Decomposed (2026-06-10): not a standalone build** —
  intentional blocking is a **`guards` brain** under
  npc-behavior (not a `BlockerBehavior`
  mixin), riding a block-substrate seam in the locomotion cascade;
  capacity + pushing are defer-til-content. See Track B §4.

### Social / perception slates

- docs/slates/tails/recognition-slate.md — per-
  viewer perception state; `DescribeApi v2` pipeline; disguise
  as Wearable shadow; salient-feature rendering.
- docs/slates/tails/social-graph-slate.md —
  per-viewer buckets (friends/foes/custom); notification policies;
  bucket-keyed display verbosity. **One *source* into the grouping
  facade** (below), not the same layer.
- docs/slates/tails/identification-slate.md —
  parallel pattern for items; experiment-based identification;
  the pedagogical seam at its richest.

The **comms / social / expression cluster** (designed as one connected
pass; built in waves):

- docs/slates/tails/senses-slate.md — the unified
  **`PerceptionChannel`** substrate: five physical senses + the **ESP
  channel family**, field/contact/network physics, the species/body
  **sensorium** (organ-gates-channel), the gestalt verb, and
  **messaging = sensing**. **Absorbs the retired sound slate** (now the
  *hearing* instance).
- docs/slates/tails/emotes-slate.md — `SoulMixin`
  natural-language emotes (+ free-form `emote`), ESP-perceived, typed-slot
  grammar, emote-only **moderation** (structural), the `Emote`-on-
  `Persistable` catalog + `SoulApi`; emoji / honorary / reactions layers.
- docs/slates/tails/comms-slate.md — the **two-
  transport** model (acoustic say/whisper/shout vs **implant ESP**);
  directed `say --to`; the `whisper`(acoustic)/`tell`(implant) split; the
  conversation primitive.
- docs/slates/tails/chat-slate.md — channels as a
  **projection over the grouping facade**; membership ≠ subscription;
  projection + override; the config block; `chat <channel>`; the Mudlog
  separation.
- [docs/subsystems/grouping.md](./subsystems/grouping.md) — the
  **`GroupApi` facade** over the four shipped providers (managed,
  MQL, contacts, channel); replaced the now-retired
  `grouping-slate.md` at the social-cluster sweep.
- docs/slates/tails/reactions-slate.md —
  emote-at-a-message + **batched aggregate-delta** broadcast (hundreds-of-
  users scale); toggle-once; tag-grouping; ephemeral runtime + warehouse
  the stream.
- docs/slates/tails/npc-dialogue-slate.md —
  conversation-not-interrogation; a **pluggable responder** (branching
  tree via prompt+engagement / scripted free-text / deferred LLM); one
  room-visible speech output.
- docs/slates/tails/access-slate.md — the
  **permission framework** filling call-security's reserved seam;
  `can(subject, action, resource)` + diegetic-first capability sources;
  the **do / see / write × circumstances** meta-shape.
- docs/slates/deferred-rpg/spoiler-slate.md — **best-
  effort** server-side fact-gating (extends the percept revelation-
  conditions); imposed + opt-in; assessment integrity flagged as a
  *separate* (assessment-system) problem.

> **Resolved:** `communication-policy-slate` (trust-tiered moderation)
> was **retired** at the docs sweep. Its stale internals (`MessageGate`,
> `effectiveTier`, emote-as-safe-fallback) contradicted shipped
> emotes/comms + the methods-only / no-registry / emotes-are-magic
> principles; its live kernel (recognition-as-security-primitive,
> trust tiers, NPC-zone trust via AccessApi) was folded into
> **comms-slate § Moderation**. The gate proper lives in **access-slate**.

### New-player & world slates

The connected new-player flow (char-gen → lounge → fast-travel →
onboarding → dorm + authoring), each thin-engine / content-heavy:

- [docs/subsystems/char-gen.md](subsystems/char-gen.md) — light
  closed-choice intake. **Wave 1 SHIPPED** (MR !49): the intake engine
  is live on the real command pipeline — `Login`-as-`CommandGiver`,
  `enroll`/`play` MVC verbs, `EnrollmentDraft` accumulator, `PersonaMixin`,
  `NameBank` suggester, 7-species roster, the cockpit char-gen phase. See
  [docs/subsystems/char-gen.md](./subsystems/char-gen.md). Deferred to
  later waves: the `records` verb (bio *editing* — its *view* role shipped
  as the `chronicle` verb), vitals/language, the lounge handoff,
  onboarding, the name sanitizer. (Breadcrumbs shipped, renamed, as the
  [chronicle](./subsystems/chronicle.md) identity ledger.)
- docs/slates/builds/onboarding-slate.md —
  learn-by-doing journey (lounge → fast-TP to campus → signs/greeter →
  dorm lobby → room → customization); the scoped-authoring on-ramp.
- docs/slates/tails/fast-travel-slate.md — the
  Eternal City Teleport Authority: a directed public-terminal network,
  **scan-to-register** credential (implant/card), on-demand teleport;
  living-infrastructure seam (maintenance/economy deferred).
- docs/slates/builds/scoped-authoring-slate.md
  — democratized, **safe, ownership-scoped** authoring (your dorm);
  GUI-first/thin-command; **(policy, validator) per-field, default-deny**.

> **Surfaced-but-deferred subsystems.** The new-player and authoring
> pass named several systems it deliberately does *not* solve, recorded
> here so they aren't re-discovered cold:
> - **Economy / currency** — catalog `make` costs, the clinic comp.
>   Multiple design sessions of its own; most of the world is
>   free/comped until it exists. (**Fast-travel fares now ship** —
>   the banking-backed transit-fare economy, conserved, split
>   city/TPA/destination — see [fasttravel.md](./subsystems/fasttravel.md).)
> - **Object condition / maintenance** — the "living infrastructure"
>   seam (terminals break down, need upkeep); illustrative in
>   fast-travel, not built.
> - **Crafting** — player-set *functional* stats within a balanced
>   envelope; scoped authoring touches no functional stats — those
>   come from vetted catalogs. (**The crafting substrate now ships** —
>   recipes, Grade quality, the knowledge ladder, repair/salvage, three
>   branches over one skeleton, see
>   [crafting.md](./subsystems/crafting.md) — with functional stats
>   derived from Material physics, never player-set; the player-set
>   sense stays deferred.)
> - **Assessment integrity** — server-side grading + don't-publish-
>   answers + assessment design; owned by the future education-vertical
>   /assessment system, *not* the spoiler slate (which is best-effort
>   experience-spoiler gating only).
> - **Campus / city-services pattern** — char-gen and onboarding lean
>   on diegetic service NPCs/kiosks (enrollment, the clinic, the
>   Teleport Authority); the reusable "service" content pattern is
>   implied, not yet abstracted.
> - **Wayfinding / signs** — Readables that direct movement; a small
>   content pattern (Readable + language + directions) onboarding needs.

### Cross-cutting

- docs/slates/tails/mixin-slate.md — broad mixin slate;
  most affordance mixins now distributed into the substrate
  slates above.
- **Affordance attribution (shipped)** — the former verb-provisioning
  slate is retired; a verb may be afforded by many source objects
  (innate `'self'`, wielded instrument, future skill / implant) and the
  source object — not a category enum — is the discriminator. Shipped:
  `CommandGiverMixin.getAffordances()` (source-preserving recency walk),
  `CommandContext.commandSource` (the affording object, threaded at
  dispatch), and the `affordances` introspection verb. Source-varied
  *rendering* rides the first build that adds a second non-innate
  source kind (skills / cybernetics). See
  [command-routing.md § Affordance attribution](./subsystems/command-routing.md).
- **Bulk substrate (shipped — thermos slice)** — continuous matter
  (liquid) as a holder attribute, NOT a Stuff: `BulkableMixin`
  interior/surface slots, the `transfer` primitive, the closure scale,
  the full MQL surface (`:b`, material keyword, `:{N unit}`), and
  fill/pour/spill/drink/sip. See
  [bulk.md](./subsystems/bulk.md). Deferred tails (mixing, gas/`sealed`,
  `Container`+`Bulkable`, amount-aware appearance) remain in
  bulkable-slate.md.
- **Wire substrate (shipped)** — response-envelope +
  mql-subscription graduated to subsystems; see **Foundation**
  above ([response-envelope.md](./subsystems/response-envelope.md),
  [mql-subscription.md](./subsystems/mql-subscription.md)). The
  slate drafts remain for design history.
- docs/adjoining-systems.md —
  catalog of unexplored subsystems (Tier 1 graduated; Tier
  2/3 remain).

### Client

- docs/slates/tails/client-cockpit-slate.md
  — affordance-first cockpit on the existing command-bus + MML wire.
  Command-bus primacy (every interaction emits a visible command,
  including the modal); click model (hover preview / click send /
  shift-click edit); server-driven modes (world / study / classroom
  / tutor) with admin `mode` override; layout sketches; panel
  inventory; content surface payload union (video / quiz /
  live-stream / classroom) with v1 = video + transcript;
  MML semantic-tag taxonomy and renderer contract;
  MQL-subscription consumer pattern; character creation as guided
  command-emitting affordances + diegetic refinement; prompt
  format; envelope rendering; mobile flagged out-of-scope but
  architecturally accommodated. Decomposes into per-track
  requirements docs at build time. Sister to
  mql-subscription-slate on
  the wire side.

- **Message rendering + prompt stack + inspection pane (shipped)** —
  all three graduated to subsystems
  ([message-rendering.md](./subsystems/message-rendering.md),
  [prompt.md](./subsystems/prompt.md),
  [card-surface.md](./subsystems/card-surface.md)); see
  **Foundation**. Message rendering shipped 2026-06 (nested-aware
  renderer, stylesheet engine + three themes, per-message-type
  templates, Discord-dialect markdown, custom URI schemes, mentions,
  `style` verb + overlay). The server `PromptApi` is live, the focus
  pane is live, and the prompt-stack client UI shipped in the
  console-foundations merge via the CommandBar's slot multiplexer
  (every Tier 1 kind renders through one surface). The message-
  rendering slate stays open for Wave 2 (layout library) + Wave 3
  (channel stylesheets); the prompt-stack slate stays open for
  Tier 2/3 kinds.

- docs/slates/tails/console-filtering-slate.md
  — sister surface to the inspection pane: client-side toolkit for
  managing the terminal scroll. Topic toggles, search,
  sender-scoped filtering, family mute/collapse, timestamps,
  compact mode, server-side `prose.verbose = brief | full` setting
  + `look --brief` flag. Principle: server always emits, client
  decides what to show — filtering is reversible without server
  round-trips. New `console.*` settings keyspace under the existing
  `EnvironmentMixin`.

---

## v1 punch list — small, concrete remaining items

Tactical work that doesn't need a slate. Pull these in
opportunistically.

- **Interactive prompt stack (Framework 11)** — *shipped*. Server
  (`PromptApi` choice / confirm / text / mqlObject / mqlMany +
  cardinality disambiguation; see
  [prompt.md](./subsystems/prompt.md)) and client (CommandBar slot
  multiplexer) both live end-to-end. Tier 2/3 kinds remain in the
  slate.
- **MQL disambiguation prompts** — *shipped* end-to-end (`onExcess:
  prompt` → `PromptApi.mqlObject` → CommandBar chip render → typed
  or chip-click response).
- **MQL sort / named-group operators** (`:sort.X`, `@@group`).
  Add when demand is real.
- **Real authoring-tier permission check** in MQL — *shipped*
  (2026-06): the `_MqlAdminFlag` stub retired, the resolver
  now consults a dispatcher-stamped `ctx.permission` snapshot
  populated from `AccessApi.isAuthor`; see
  [access.md](./subsystems/access.md). Per-result resource
  targeting (each `prop:`/`mixin:`/`class:`/`template:` filter
  hiding non-author-readable matches) remains as the next
  authoring-tier MQL build.
- **Markup language semantic tags + client renderer** — extend
  MML with `<command>` / `<direction>` / `<item>` / `<exit>` /
  `<npc>` / `<player>` / `<quantity>` and formal tags
  (`<color>` / `<size>` / `<link>`). Foundation for clickable
  links and richer rendering.
- **Look fallback for non-Visible rooms** — current "You see
  nothing special." reads wrong for plain locations like the
  void.
- **Model piping** (PowerShell-style) — foundational for
  scripting; medium.
- **Utility APIs** — `StringApi`, `TimeApi`, `ObjectApi`,
  `CallstackApi`, `FileApi`, `AssertApi`. Take on demand.
  `MudlogApi` exists but is incomplete.
- **Recognition / viewer-aware presentation** — implements the design
  from recognition-slate.md.
  Wave 0 (the viewer-blind `Stuff.getPresentation()` baseline +
  retirement of the static `DescribeApi`) **shipped**; what remains is
  the viewer-aware composition pipeline: per-viewer recognition /
  identification memory, `getDisplayParts`, MML-aware decoration.

---

## Substrate buildout — slate implementation

The early dependency stack is **shipped**: Quantities → Embodiment
→ Locomotion → Activity (see Foundation + the subsystem docs). The
remaining substrate is sequenced as **Track B** under *What's next*
above (access → senses → social cluster → collision → recognition
family → augmentation/npc-dialogue), plus the **Race follow-on**
slices below.

One sequencing note worth keeping: the **recognition family**
(recognition + identification + social-graph) ships best as a
*unit* — the slates compose tightly (per-viewer state + DescribeApi
v2 + buckets), and dispersed shipping creates hard-to-test partial
states. (The persistence-framework upgrade the per-record stores wanted
has since shipped as the self-persistence spine — see
[persistence.md](./subsystems/persistence.md).)

---

## Race subsystem follow-on

V1 shipped the substrate. Deferred work, sequenced as content
demands:

- ~~**Death / resurrection flow**~~ — **SHIPPED 2026-08-01**
  ([mortality.md](./subsystems/mortality.md)): the dying clock, the
  single `ConditionApi.die` transition, the corpse as a forensic
  `Creature`, the shade, and `reembody` + the `passage` floor. Still
  open: the underworld content, the re-embodiment service, and the
  recuperation model (the floor ships one crude diminishment).
- **Instantiate the `Condition` catalogue at boot** — surfaced by the
  mortality build's end-to-end pass. Condition seeds are inserted as
  template rows and nothing clones them into Ideas, so
  `findByTemplatePath` answers null for every condition in a running
  world and all authored `Condition` behavior (observable signs, names,
  progression, `toxinBehavior`) is inert. Every consumer tolerates the
  null, so nothing fails loudly. Small build; unblocks the disease and
  medic verticals. See [vitals.md](./subsystems/vitals.md).
- **DietApi + Edible + Portable** — material toxicity authored
  but no consumer reads it. Needs eater-side diet check.
- **Per-Detail materials and tissue authoring** — v1 is
  bulk-only. Needed for tissue-zone seams (eye, wing, hand).
- **Genetics** — alleles, inheritance, mutation, evolution. A
  sub-subsystem of its own.
- **Per-individual variation** — feature mixins for the
  unique-individual layer above species.
- **Sleep / circadian** — per-species rhythm; status mixins
  for sleeping / resting.
- **Aging** — life stages, species lifespans, per-stage
  property changes.
- **Polymorph** — runtime body-plan swap. Slot map
  reconciliation across body-plan changes.
- **Character-creation UI** — currently no UI for picking
  species, sex, gender, body-plan-derived options.

---

## Adjoining systems still in queue

From docs/adjoining-systems.md. Tier
1 graduated to slates; remainder by tier:

**Tier 2** — extends established patterns:

- #4 Scent and persistent traces — **smell is now a channel in
  senses-slate** (the `PerceptionChannel`
  substrate); the remaining work is the *persistent-trace* half (trails
  that linger), which needs activity emission hooks (deferred in
  activity-slate) and pulls on #9.
- #6 Visibility-within-room — what's visible at varying
  containment depths; partial absorbance into DescribeApi v2
  via recognition-slate.
- #7 Memory of observed events — partially absorbed by
  recognition + identification's per-viewer stores; broader
  Witness-pattern memory is the unfinished part.
- #10 Activity layer — non-locomotion sustained tasks (read,
  forge, brew); largely covered by activity-slate; specific
  content (a brewing recipe, a reading flow) lands as the
  content asks.

**Tier 3** — peripheral / forcing-function-driven:

- #8 Multi-actor coordination — lift-the-log-together,
  carry-the-stretcher, two-player levers. Stresses
  activity-slate's single-actor model.
- #9 Persistent location state — bloodstains, footprints,
  soot. Temporal traces; pulls on #4.
- #11 Heat as physics channel — the **`PerceptionChannel`
  generalization is now realized in
  senses-slate** (thermoreception is a
  channel); remaining work is heat's *physics* (propagation / sources /
  Quantity in Kelvin), parallel to light and sound, plugging into the
  established channel shape.
- #12 Pedagogical seam — largely absorbed by quantities-
  slate; remaining work is content-team integration.

---

## Platform / production

Required for v1.0 ship. Most depend on the substrate slates
landing first.

- **Templates, mods, and isolated-vm sandboxing (Framework
  13)** — mod base class (Content / Capability / Full), mod
  registry, dependency loader, `isolated-vm` integration,
  bridged whitelisted Apis, resource limits (CPU / memory /
  timeout), monitoring hooks. The runtime-model
  Tier 2 isolation discussion
  is the framing. *The **Content** corner has shipped its first
  build as **content packs** — pure-data, no-code deliverables with a
  reconcile installer (see
  [content-packs.md](./subsystems/content-packs.md) /
  content-packs-slate.md);
  Capability / Full (new-code mods, needing `isolated-vm`) remain.*
- **Persistence framework upgrade** — fine-grained per-record
  access patterns. Recognition + identification + social-graph
  stores need it; current `Persistable` is whole-document. May
  fold in a parallel "social/memory store" using MongoDB
  collections directly, with its own schema and indices.
- **Idle eviction for Stuff lifecycle** — **shipped** as the residency
  substrate ([residency.md](./subsystems/residency.md)): a real-time
  sweep lets abandoned `Stuff` self-evict via the `canEvict` hook
  (default-cull, override-to-veto), recency fed by dispatch-touch +
  presence, observe-first. The game-time **reset** sweep is the deferred
  sibling in the same home.
- **Guest accounts** — **shipped** as the anonymous-guest path of the
  client shell (see [client-shell.md](./subsystems/client-shell.md)):
  a `/auth/guest` ephemeral principal, a randomized throwaway avatar
  minted on Enter with a reserved-word NameBank name, destroyed on
  disconnect, persisting nothing. Lowered the barrier to first-time
  exploration. *Remaining*: abuse controls (rate-limit / per-IP /
  captcha) hang off the single `mayMintGuest` gate when needed.
- **GraphQL admin API** — `type-graphql` schema, resolvers
  over running game state for inspection / dashboards.
- **Production hardening (Phase 10)** — test coverage to >80%,
  integration / E2E flows, sandbox escape tests, MongoDB
  connection pooling, message batching, memory-leak audit,
  load testing, error boundaries, admin commands.
- **Deployment infrastructure** — Docker image, AWS CodeDeploy
  + Parameter Store + Secrets Manager + S3 + EC2; GitLab CI;
  health checks. Old PLAN.md AWS section has the spec.

---

## Client UX

Canonical client design surface is the cockpit slate:
docs/slates/tails/client-cockpit-slate.md.
It covers the v1 cockpit (layout, click model, modes, panel
inventory, MML semantic tags, MQL-subscription consumption,
content surface, prompt line, envelope rendering, character
creation) and names what's deferred. Per-track requirements docs
decompose from the slate at build time.

Long-term opportunities explicitly out of v1 cockpit scope (the
slate flags them as separate projects):

- **Visual map generator** — 3D map rendered from the spatial
  subsystem. Own project, own slate; cockpit reserves the layout
  slot but does not build it.
- **Content illustrations** — **shipped** as the media subsystem
  (see [media.md](./subsystems/media.md)): model-driven *offline*
  generation in one house style, `Visible.illustration` key → MQL
  projection → client render (inspection pane + char-gen portraits),
  `MediaAsset` provenance, S3 serving. *Remaining*: on-demand /
  in-CMS generation with a quota system, external embeds
  (Twitch / video), and the cohesion post-process pass.
- **Dedicated content CMS** — Wave 1 shipped: a REST-only CMS surface
  of the SPA (`?surface=cms`) with a unified file/template explorer and
  a lazy-loaded Monaco code editor over the existing source/template
  backends, dev-tier and context-gated (see
  [cms.md](./subsystems/cms.md)). **Wave 2 (the Studio) shipped**: the
  mixin-aware composition surface — a schema-driven, `@authorable`-derived
  content-editor form, the blueprint catalogue, and the class scaffold/commit
  bridge across the wizard-lockdown code boundary (see
  [studio.md](./subsystems/studio.md)); it also lit the first authoring-
  intelligence catalogs (mixin particles + named blueprints). *Remaining*
  (per cms-slate.md +
  authoring-intelligence-slate.md):
  per-type / bespoke content editors + the zone/map canvas, the
  external-editor/LSP / engine-typed IntelliSense path, drafts/publish + the
  law==code forums-review gate, and versioning.
- ~~**Mobile cockpit**~~ — **shipped**. The chrome half landed
  2026-08-12 (Build C, client-rebuild Wave 1); the **play surface**
  landed with waves 2.5 + 4: one card per thing you look at, stacking
  and ageing, one strip of named views, the radial and the prompt
  strip, all at both form factors. See
  [card-surface.md](./subsystems/card-surface.md) and
  [client-shell.md](./subsystems/client-shell.md).

  ⚠ Two pieces named here were deliberately NOT built, and the reasons
  are recorded rather than deferred by omission: **routed feeds** were
  retired (a routing stamp applied at delivery cannot re-sort history,
  and the frame store does not persist it — so every backfilled frame
  fell to `world`), and the **focus chain** became a detail trail
  scoped to one card, because the card stack is the history a
  breadcrumb used to stand in for.

---

## Aspirational / long-term

- **Domain mods** — Education (adaptive learning, course /
  quiz events), Retail, others.
- **AI-driven NPCs** — LLM-backed faculty, staff, student NPCs
  as a Capability Mod.
- **Modding marketplace / community content** with the
  sandboxed mod API.
- **In-game scripting** for users (sandboxed).
- **Web forms for complex commands** (crafting UI, character
  sheet) and graphical elements (avatar art, room
  illustrations).
- **LMS integration** — sync, progress tracking, adaptive
  content.
- **Distributed deployment** — sharded zones, cross-server
  social-graph federation.
- **Phase 11+ persistence** — full `Thing` persistence with
  location reconstruction, advanced template diffing.

---

## Suggested order

The near-term order lives in **What's next — two parallel tracks**
(top of the doc): Track A (client/cockpit) and Track B (server
substrate) run concurrently, each with its own six-item sequence.
Beyond those near-term items:

1. **Tracks A + B in parallel** — the two near-term sequences. The
   v1 punch-list tactical items (markup tags, look fallback, MQL
   extensions, utility APIs) land opportunistically alongside,
   without blocking either track.
2. **New-player flow** — char-gen → onboarding → fast-travel →
   scoped-authoring, drawing the server halves from Track B and the
   GUI/journey from Track A as both mature.
3. **Race subsystem follow-on slices** — pull as content needs.
   Death / DietApi / tissue early; genetics later.
4. **Mods + isolated-vm + persistence upgrade** — v1.0 platform
   work. Significant lift; start once substrate feels stable. The
   per-record persistence upgrade probably lands with the
   recognition family.
5. **Production hardening + deployment** — once mods exist.
6. **Aspirational** — opportunistic.

---

## What got skipped or absorbed

For audit. Items from previous roadmaps that are done, absorbed,
or no longer load-bearing:

- **Phase 5 ("Communications")** — absorbed; `say` / `tell` +
  `Sensor` / `Vocal` shipped.
- **Phase 6 ("Extended Object Model")** — absorbed; `Thing`,
  `Detailed`, `Propertied`, `CartesianLocation` all in tree.
- **Phase 8 ("Advanced API Layer")** — partially shipped; rest
  is the "Utility APIs" punch-list item.
- **Light & Boundary subsystem** — shipped. The Light value
  object, propagation walk, per-viewer perception, and the
  Boundary substrate (Window, Door retrofit) all landed.
- **Race subsystem v1** — shipped. Material substrate, Clade,
  BodyPlan + Species, OrganismMixin, SexedMixin, SpeciesApi,
  animacy gating.
- **Event System** — shipped. EventApi + Witness pattern +
  EventRegistry + lifecycle hooks; class-per-event vocabulary
  added for the subscription substrate.
- **Client/server wire substrate** — shipped, graduated from
  slates to subsystems: response-envelope, mql-subscription
  (live-state diffs), prompt (`PromptApi` + cardinality
  disambiguation), and the inspection pane. The cockpit's
  foundation; remaining client work is Track A.
- **Module hot-reload** — shipped. HotReloadApi + admin
  `reload` command + clone integration.
- **AliasMixin** — shipped. Per-character verb aliases.
- **Shell tooling** — shipped. Workspace + Author + Perceiver +
  Environment mixins, the verb suite, HomeZone, spawn-shape.
- **Bootstrap subsystem** — shipped. `BootstrapManager` for
  ordered system-singleton creation.
- **Marshaller framework** — shipped. Custom serialization
  escape hatch.
- **`MarkupApi`** — PLAN.md spec'd a server-side helper class;
  current code calls into `api/mml.ts` directly. Decision
  pending: keep direct or formalize wrapper. Punch-list item.
- **VM2** — settled on `isolated-vm`. No action.
- **Phase 6-8 client features** (split-pane, tabs, mini-map,
  sound) — rolled into "Client UX."
