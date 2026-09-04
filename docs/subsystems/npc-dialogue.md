# NPC dialogue (Wave 1) — the branching-tree responder

How players hold conversations with NPCs. This build delivers the
**responder seam** — `talk to <npc>` routes you to a pluggable responder
behind the NPC, and the responder replies through the ordinary,
room-visible speech path — plus the **first responder: the branching-tree
responder**, composed from shipped substrate (`PromptApi` choices, the
`SustainedEngagement` framework, the Scene-composed speech path).

Dialogue is multiplayer-native by construction: the wheel of choices is
private to the chooser, but the resolved line is *spoken aloud* and heard
by the room — **choosing is interior, speaking is exterior**. **The NPC
leads**: you engage, the NPC speaks the guard-selected root beat, your
response choices follow.

Rides the behavior substrate ([behavior.md](./behavior.md)) — the
responder is a **brain** — and consumes comms ([comms.md](./comms.md)),
prompt ([prompt.md](./prompt.md)), and activity
([activity.md](./activity.md)). Phase 2 of the
Dave's Bar slate ("they talk").

## The responder seam

```
talk to <npc>  →  TalkController
                    → narrow target to a Behaved host
                    → find its `engage`-triggered BehaviorSpec
                    → resolveExportSync(spec.brain, 'brain')
                    → brain.open({ player, npc, config, interactive })
                          → new DialogueConversation(...)  (the live state machine)
```

The controller does **not** walk the tree — it hands off to the brain's
`open` and returns. The brain stays the only code that knows the tree
format; a second responder (`intent-dialogue`, deferred) is a drop-in
implementing the same `open` shape, no contract change.

- **`open` seam** (`lib/behavior/brain.ts`): `BrainStatics.open?(args:
  DialogueOpenArgs): Promise<DialogueOpenResult>` — `{ok:true}` or
  `{ok:false; reason:'no-tree'|'busy'|'no-viewer'}` so the controller
  renders the right decline prose. Marked `@hook`; only dialogue brains
  implement it. Distinct from `act` (the trigger entry).
- **The `engage` trigger** (a new `ParsedTrigger` variant
  `{source:'engage'}`): wires **nothing** — no cadence timer, no witness
  dispatch. It exists only so (a) `getBehaviors()` surfaces the tree to
  the `talk` controller, (b) the brain path is warmed at wire time (so
  `resolveExportSync` hits), and (c) the host reads as conversational for
  discoverability. The requirements forbid speech-triggered trees this
  build; the brain is reached **only** through `open`.
- **The verb** (`cmd/social/talk.yaml` + `platform/idea/cmd/social/Talk
  Controller.ts` + seed): `verbs:[talk, converse]`, one positional
  `target` (`type: object`, `scope:['$focus','reachable']`,
  `prepositions:[to]` so both `talk to mara` and `talk mara` parse),
  `requiresAnimate`. Emits **no player line** on open (the NPC leads).

## The tree format (`lib/npc/tree.ts`)

A tree is the **`config` blob** of the NPC's `tree-dialogue`
`BehaviorSpec` — pure declarative data persisted inside the NPC template's
`data.behaviors[].config` in the existing **`domain`** collection
(hydrated by the normal `PersistentHydrator`). **No `dialogue`
collection, no `DialogueTree` Stuff/Document** — the interfaces describe
the blob's shape; nothing is ever instantiated. Because `config` is
`Record<string, unknown>`, the shape is NOT TS-checked at hydration —
`DialogueTreeSchema.validate` is the only structural guard.

```ts
interface DialogueTree { entry: DialogueEntry[]; nodes: Record<string, DialogueNode>; }
interface DialogueEntry { node: string; guard?: DialogueGuard[]; }   // first passing wins
interface DialogueNode { beat?: string; choices?: DialogueChoice[]; terminal?: boolean; }
interface DialogueChoice { line: string; guard?: DialogueGuard[]; to?: string; effects?: DialogueEffect[]; }
interface DialogueGuard { fact: GuardFact; op: GuardOp; value: string|number|boolean; }
```

- **Guard fact namespace** (fixed, validation-array-driven): `regard`
  (`npc.regardFor(player)`), `trait:<axis>`
  (`npc.traitPosition(axis).position`, axis ∈ the disposition
  roster), `time:hour` (world hour-of-day), `state:<key>` (ephemeral
  conversation scratch), `position:<organization path>` (whether the
  interlocutor holds a non-exited position there — an employer's
  *"looking for work?"* branch guards on it being `false`; the lounge's
  Dave is the first consumer). Operators: `eq/ne/gt/gte/lt/lte`.
- **Intrinsic effect verbs** (conversation primitives, hardcoded — they touch
  the conversation's own machinery): `set-state`, `regard` (delta NPC→player,
  persists), `say`, `emote`, `goto`, `end`, **`dispatch`** (the "NPCs do their
  jobs" seam — below).
- **Domain effect verbs** (genuinely extensible — no reopening the format): a
  domain registers a handler in the **`DialogueEffectRegistry`**
  (`lib/npc/DialogueEffects.ts`, `{validate?, apply}`), and both
  `validate`/`apply` here delegate to it — so the generic dialogue substrate
  never imports the domain (consumer → substrate). Banking's **`bank-circle`**
  (enrol the interlocutor into a corpo's Circle) is the first, registered from
  `lib/banking/BankDialogueEffect.ts` at boot.
- **`DialogueTreeSchema.validate(tree): string[]`** — collect-all-errors,
  never throws, tolerates `unknown`. Checks dangling `entry[].node` /
  `choice.to`, unknown facts/axes/operators, unregistered effect verbs.
  A value-object class owning the operations (the `EmoteGrammarRunner`
  precedent) — satisfies `lib/` export discipline (no free-floating
  `validateTree`).

## The `dispatch` effect — NPCs do their jobs

The **`dispatch`** effect (`{verb:'dispatch', command}`) is how a
conversation performs a real world-action: the NPC **runs a command as
itself** through the command bus. It is the reusable seam that lets a service
NPC *do its job* — Katie the dorm super assigns you a room, a banker opens an
account, an employer hires you — without a bare player-facing verb. `$player`
in the command renders to the interlocutor **by identity** — `#<stuffId>`, the
viewer-free MQL seed — never by name: the NPC names people by what it
*recognizes*, and a stranger it has not been introduced to is "a human" to it
(Dave's `appoint $player …` found "no such person" until this); `applyEffects` narrows the NPC to a
`CommandGiver` and calls `CommandApi.forceCommand(npc, text)` from the
detached loop's context.

**The security model (load-bearing).** The command runs **as the NPC, bounded
by the NPC's own authority**, through the **ordinary dispatch chain**:
affordance match → MQL resolve → validator preload → **validators** →
`execute()`. A dispatched NPC command is gated exactly as a typed player
command is.

> ⚠⚠ **CORRECTED 2026-08-04 — this section previously said the opposite.**
> It read *"`forceCommand` stamps `forced:true`, which bypasses the
> affordance/YAML-validator gates."* **That is false.** `forceCommand` only
> sets the flag (`giver.executeCommand(text, {...opts, forced: true})`);
> `forced` is read in exactly two places — the server-authoritative
> input-mode prefix (skipped, since a dispatch carries no `barId`) and the
> Command frame's metadata stamp. **Nothing in the resolve/validator path
> consults it**, and `runValidators` runs verb-level validators
> unconditionally.
>
> **Verified by experiment, not by reading** — the same gated verb
> dispatched forced and un-forced produces the *identical*
> `validator-failed` note. ⭐ Note the trap that hid this: validators are
> resolved onto `_resolvedValidators` **only by `CommandApi.preloadAll`**,
> so a test harness that calls `getCommand` without preloading sees
> declared-but-unresolved validators and **silently skips every one of
> them** (`runValidators` guards on `if (command._resolvedValidators)`).
> A probe without `preloadAll` shows forced and un-forced both reaching the
> controller and looks like proof of a bypass. It is not.

**So `forced` grants an NPC nothing.** The affordance set still applies (the
verb must be afforded to that NPC) and every YAML validator still runs.

⚠ **This matters more than a documentation nit**, because content was
authored against the false claim: a verb whose authorization was moved to
`execute()` *on the belief that the YAML gate would be skipped* still has
its YAML gate, and will refuse the NPC. **`provision` is exactly this
shape** — it carries verb-level `requiresWizard`, Katie is deliberately not
a wizard (`KatieProvisioning.test` asserts `isWizard(katie) === false`), and
her intake dialogue dispatches it. See [residence.md](./residence.md).

⭐ **Controller-level authorization is still right**, just for the ordinary
reason rather than the stated one: **defence in depth.** A controller should
not rely solely on its YAML, and `execute()` is where a *relational* rule
("an agent of this parcel's owner") belongs, since a validator cannot see
the bound model. Provisioning's `isDormsAgent` is a good gate. It is simply
**not the only** gate.

The command string is **not** statically validated (the scripting precedent
— bounded at runtime by authority, not at save time); `DialogueTreeSchema`
only checks the effect is well-formed (`dispatch` present with a string
`command`).

**Best practice (content development).** A player-facing capability gets a
**diegetic front** — an NPC (or object) doing its job — never a bare typed
verb; the raw verb stays an operator/debug seam. The **NPC is the
authorization principal** (its role/membership is why the action is allowed),
and `dispatch` is the reusable primitive that wires the two together. See
[residence.md](./residence.md) (Katie) for the first consumer.

## The conversation engagement (`lib/npc/DialogueConversation.ts`)

A controller's `CommandContext` dies after one turn, but a tree walk
spans many picks — so the live state machine lives on a
`SustainedEngagement` instance (in the `SchedulerRegistry` active set,
keyed to the NPC actor — exactly where `RespirationDrain` lives), NOT on
the controller or the stateless brain.

- **Both-sides slot hold.** An engagement has one actor and sets slots on
  that actor only, so a conversation is **two** engagements:
  `DialogueConversation` holds `voice`+`attention` on the NPC, and a
  companion `DialoguePartnerHold` holds them on the player. They
  reference each other; ending or aborting either releases both
  (mutual-cancel is idempotent — `SchedulerApi.cancel` of an
  already-deregistered engagement is a no-op). Both classes land in the
  activity-class dispatch index automatically at `start()`
  (capture-at-start), which `SchedulerRegistry` consults for
  `onAbort`/`getHost` dispatch.
- **The detached loop.** `open` validates, picks the guard-selected
  entry, starts both engagements, then `launch()`es the choice-await loop
  via `ScheduleApi.schedule(0, …)` — **not** `ExecutionContextApi.runRoot`
  directly (domain code may not plant call frames; `ScheduleApi` wraps
  the callback in `runRoot` for us). The loop runs in a fresh
  execution-context root so its `await`s — and the continuations that
  resume when the player picks — outlive the originating command frame.
- **The loop:** emit `npc.say(node.beat, player)` (directed, room
  overhears); build the guard-filtered choice list; `await
  PromptApi.choice(interactive, label, choices)`; on pick →
  `player.say(choice.line, npc)` (the exterior half), apply effects in
  order, advance to `choice.to`/`goto`; terminal/no-choices → end.
- **Guards / effects** are private methods (they own the conversation
  state the facts read). Unknown fact fails closed (the save-gate already
  rejected it; this is defense-in-depth).
- **End / abort.** `onAbort` is the single teardown: stop the loop,
  `PromptApi.cancelAll` the in-flight wheel, cancel the partner hold.
  Reached by the terminal node, `prompt cancel`/disconnect
  (`PromptCancelledError` → loop ends), host-destruction (the registry's
  `StuffDestructed` sub), or **room-leave** (a co-presence check each
  loop turn + `reviewPresence()`, poked from `BehavedMixin.handleMessage`
  on a perceived movement frame). The NPC's ambient cadence brains resume
  next tick once `voice`/`attention` free (they declare `requiresFree` —
  see [behavior.md](./behavior.md)).

**Ephemeral only**: the `scratch` bag dies with the engagement
(never persisted). Cross-conversation warmth lives in **regard**
(persisted via the belief store), never here.

## Discoverability — the `InstanceContributor` seam

A tree-bearing NPC must afford `talk` to nearby givers; a silent one must
not. `commandContributions` is **static/class-level**, so it can't be
conditional on per-instance tree data, and `getAffordances()` is a pure
push-stack read (no live hook). So the affordance infra gained a
first-class **per-instance** seam:

- **`InstanceContributor`** (`api/command.ts`, `@hook`):
  `getInstanceContributions(): CommandContributions`. The
  containment-delta machinery (`CommandLogic.applyContainmentDeltaImpl`)
  merges it with the static contributions at the `environment`/`peers`
  push sites (`collectBucketDefsForInstance`). Because it rides the
  ordinary push/pop/reset movement lifecycle, late-arrival, departure,
  and mover relocation are all handled with no extra hooks.
- **`BehavedMixin.getInstanceContributions`** returns `social/talk.yaml`
  (environment+peers) **exactly when** the host carries an `engage`
  spec — reading the persisted `behaviors` (live before placement), so
  "has a tree ⇒ affords talk" is automatic and silent NPCs never afford
  it. The examine cue is authored prose on the NPC's long description
  (content, no code). `MixinApi.isBehaved` added for the controller's
  narrow. See [command-routing.md](./command-routing.md).

## CMS save-gate

`CmsLogic.validateBehaviorPaths` (the `_writeContent` gate) additionally,
for any spec whose resolved brain `label === 'tree-dialogue'`, runs
`DialogueTreeSchema.validate(config)` and throws `CmsError('invalid', …)`
on a dangling node target / unknown effect verb — caught at author time,
not conversation time.

## Auto-introduce — knowing who you're talking to

Speech renders the speaker viewer-relatively
(`describeFor` via `Mml.actor`), so an un-introduced NPC shows
by description ("a watchful dwarf"), not name. The fix is **real
recognition via auto-introduce**, not a name bypass:

- **`SoulMixin.introduceSelf()`** — the shared, modality-neutral
  introduce act (the introduce scene + `learnIdentity` to all in range).
  Introduce is a Soul act on every Character.
  `recognizes(viewer, subject)` is the gate (don't
  re-introduce to someone who already knows you).
- **NPC (intrinsic):** the **`introduces` brain** (arrival witness) — on
  a newcomer arriving, if they don't already know the host, the host
  introduces itself. Wired into the warm bartenders; a secretive NPC
  (Sloane) simply omits it. NPCs have no settings store, so their intent
  rides behavior data — this is the only intrinsic seam.
- **Player (opt-in):** the **`social.autoIntroduce`** setting (SoulMixin,
  default off; PCs compose `EnvironmentMixin`) + **`Mobile.autoIntroduce
  OnArrival()`** (fired alongside `autoSenseOnArrival` in
  `traverse`/`teleport`): an opted-in mover introduces to a room that
  doesn't know them, and opted-in occupants introduce to a newcomer. It
  reads the setting, so NPCs (default off) are untouched.
- The **pinned dialogue prompt** names the speaker recognition-aware:
  `Mara: What'll it be.`

See [belief.md](./belief.md) (recognition / `learnIdentity`) and
[emotes.md](./emotes.md) (SoulMixin).

## Client UX (`packages/client/.../CommandBar.tsx`)

- **Pinned prompt context.** The active prompt's label (for a tree, the
  NPC's current beat, recognition-named) is pinned directly above the
  choice chips so it stays put while the room feed scrolls — shown in
  full (wraps). The cramped one-line slot-picker pill **left-truncates**
  (keeps the operative end, ellipsis at the start, via `direction: rtl`).
- **Choice chips** are numbered (`1. <line>`); hovering one previews its
  number in the command bar and typing that number + Enter sends the same
  response as clicking — the GUI→CLI on-ramp (every clickable previews
  the command it sends). `confirm` chips preview `yes`/`no`.

## The `lib/npc/` home

`lib` is organized by scope. `lib/shell/` is PC-only, `lib/character/` is
the shared PC∪NPC union; `lib/npc/` is the missing symmetric third — types
that apply **only to NPCs**. `NPC.ts` relocated here (Wave 0,
`/lib/character/NPC` → `/lib/npc/NPC` — a class-path re-key requiring a
one-time operator Mongo migration, flagged in the MR), alongside
`tree.ts` and `DialogueConversation.ts`. The `tree-dialogue` /
`introduces` brains stay in `lib/behavior/` (path-pinned by the
Module-Categories taxonomy). `Crafter` (`extends NPC`) is a natural
follow-on relocation, not done here.

## Deferred seams

- **Scripted `intent-dialogue` responder** + the `addressed` /
  `handleMessage` trigger + the implant `tell` (remote) entry — slate
  Wave 2. The `open` seam accommodates it with no contract change.
- **LLM responder** — slate Wave 3.
- **Persistent per-relationship state** — the social-graph build (regard
  already carries cross-conversation warmth).
- **Multiplayer tree participation** beyond overhearing — slate Wave 3.
- **Complex guard/effect expressiveness** (computed predicates, scripted
  beats) — the scripting build; the registered verb/fact sets are the
  seams quest/advancement extend.
- **Auto-introduce edges** (same class as `greets`): logging in already
  standing in a room, and an NPC arriving into a room with a stationary
  player, don't fire the arrival introduce.

## History

Built on `feature/npc-dialogue-build` (commits `830c43b2..baafab84`): Wave
0 NPC relocation, Wave 1 tree format, Wave 2 conversation engine + talk
verb + the `InstanceContributor` affordance seam, Wave 3 CMS save-gate +
seed cast (Mara). Post-build UX iteration added the pinned prompt
(named, left-truncated pill), numbered choices with hover-preview, the
speech-brain `requiresFree` fix (quiet while engaged), and auto-introduce.
