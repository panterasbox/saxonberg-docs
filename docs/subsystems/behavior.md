# NPC behavior

The behavior subsystem is the automation layer behind authored
(non-player) characters — and the **first real consumer** of the
shipped-but-inert [activity framework](./activity.md). Almost everything
an NPC does reduces to one shape: a host carries a declarative
**`behaviors:` data list**, and a swappable **brain** decides what to
emit when a trigger fires. The NPC holds *data*; the brain holds *logic*;
emission rides the channels we already have (speech, emote, locomotion).

Wave 1 ships the `Behaved` mixin, the path-resolved brain category, the
cadence + perception triggers, engagement-slot contention, the thin
`NPC` class, and seven canned brains. Dialogue brains, the
`scripted-behavior` brain, the LLM brain, and traits are later waves —
this doc notes the seams they plug into.

## The model

```yaml
# an NPC template — behavior is pure data, no code
class: /lib/npc/NPC
data:
  behaviors:
    - { brain: /lib/behavior/shifts, trigger: cadence:60s,  config: { behindBar, offstage } }
    - { brain: /lib/behavior/idles,  trigger: cadence:300s, config: { pool: [...] } }
    - { brain: /lib/behavior/greets, trigger: arrival,      config: { lines: [...] } }
```

A **behavior spec** (`BehaviorSpec`) is `{ brain, trigger, config }` —
the brain is a *path* to a code module, the trigger names *when*, the
config is the brain's data. The list is a persistent field on
`BehavedMixin`; at `postRegister` the mixin reads it, path-resolves each
brain, and wires each spec to its trigger substrate. Adding behavior is
authoring data — no code, no subclass.

## Brains are a module category (`export const brain = class {…}`)

A brain is **stateless strategy code** — not a `Stuff`. Like an
`Engagement` is a plain object (see [activity.md](./activity.md)
decision #1), a brain is transient logic with no persistence, identity,
proxy, or lifecycle. It lives on the filesystem under `lib/behavior/`
and is *code*, not content cloned into the world.

A brain module's **sole concept-export** is:

```ts
// lib/behavior/patrols.ts
export const brain = class {
  static label = 'patrols';
  static claims: readonly EngagementSlot[] = ['body'];
  static requiresFree: readonly EngagementSlot[] = ['attention'];
  static async act(ctx: BrainContext): Promise<void> { /* decide + emit */ }
} satisfies BrainStatics;
```

`brain` is a **named class-expression**, not a plain object. This is
load-bearing: the hot-reload registry only retains class-like exports
(`HotReloadApi`'s `#extractClassLikeExports` keeps `typeof === 'function'`
exports with a prototype), so a plain `const brain = {}` would be dropped
and the per-invocation re-resolve seam would find nothing. Metadata and
the `act` entry live as **statics** so they read without instantiation
(the CMS palette, the contention wiring).

`BrainStatics` is the static-side contract:

| Member | Role |
|---|---|
| `label` | Display name / engagement type tag |
| `claims?` | Engagement slots the brain occupies while acting |
| `requiresFree?` | Slots that must be free for the brain to proceed |
| `presenceGated?` | Default `true`; `false` to fire even with no audience |
| `act(ctx)` | The entry point the framework invokes when a trigger fires |

`act` receives a `BrainContext`:

| Field | What |
|---|---|
| `host` | The NPC (the actor — emission is `host`-receiver, never an argument) |
| `config` | The spec's `config` blob |
| `state` | Per-(host, spec) runtime scratch bag the framework owns — patrol index, etc. Not persisted; brains stay stateless. |
| `perceived?` | For witness triggers: `{ frame, subject? }` — the perceived frame + the resolved acting subject |
| `trigger` | `{ source: 'cadence' \| 'witness', raw }` |
| `say` / `emote` / `emoteFree` | Host-bound emission helpers (safe no-ops if the host lacks Vocal/Soul), so brains carry no mixin-narrowing boilerplate |

## Path resolution + HMR

A brain is referenced by a *logical path* (`/lib/behavior/patrols`) and
resolved through `StuffApi`:

- **`StuffApi.resolveExport(classPath, exportName)`** — async; validates
  the path, maps it to the absolute fs path (the same logical→absolute
  mapping `clone` uses), and resolves the named export from the
  hot-reload registry, **warming** it (lazy `reload`) on a cold miss.
  Returns `null` (never throws) for an invalid / frozen / missing
  export. `BehavedMixin` calls this once per spec at wire time; the CMS
  save-gate uses it to validate brain paths.
- **`StuffApi.resolveExportSync(classPath, exportName)`** — synchronous,
  no lazy warm; a registry-map hit. `BehavedMixin` calls this **on every
  fire** to re-resolve the current brain class.

Re-resolving by path per invocation is what makes brains hot-reloadable:
`Behaved` stores only the path string — **never a captured brain
reference** — so editing a brain and reloading its path means the live
NPC's *next* action runs the new code, with no re-spawn. This is the
controller `clone-per-execution` HMR pattern (see
[hot-reload.md](./hot-reload.md)), applied to a plain module: cheap
because resolution is a map lookup, where a `Stuff` singleton would need
an explicit `dest` to swap the cached instance.

## Triggers: cadence + witness — no new events

A trigger is a thin selector over **two sources**. State conditions ("at
night", "my shift") are **guards inside brain code** (e.g. `shifts`
reads `WorldClockApi`), never a third source. Crucially, the trigger
surface emits **zero new global events and subscribes to zero global
event buses of its own** — everything an NPC reacts to, it already
perceives, or it is a timer.

- **cadence** (`cadence:Ns`, also `ms`/`m`) → a **jittered** recurring
  `ScheduleApi` timer (±25% per fire, so a room of NPCs never ticks in
  lockstep). Cadence fires are **presence-gated** by default: skipped
  when no player is in the host's room (an empty bar sleeps). A "player"
  is a room occupant that is a `Sensor` and not itself `Behaved`.
- **witness** → the host's own `SensorMixin.handleMessage` perception
  stream (`BehavedMixin` overrides it, super-chaining so default
  delivery is preserved; only meaningful on `Sensor` hosts). Each
  perceived frame is dispatched to brains whose trigger topic matches:

| Alias | Frame topic (prefix) | Subject |
|---|---|---|
| `arrival` | `act.move` | new room occupant (delta vs a per-host seen-set) |
| `departure` | `act.move` | departed occupant (delta) |
| `emote` | `act.emote` | recovered via `meta.commandId` → `ReactionApi.actInfo` |
| `speech` | `speech.` | same (available; `reacts` uses `emote`) |

`arrival`/`departure` compute the room-occupant delta against a
per-host seen-set seeded at wire time (so spawning doesn't "greet"
everyone already present). `emote`/`speech` recover the acting subject
from the act registry — the same speaker-recover seam renown uses.

A third trigger, **`engage`** (`{source:'engage'}`), wires **nothing** —
no timer, no witness dispatch. It marks a spec whose brain is reached
only through the **`open` responder seam** (`BrainStatics.open`, a
`@hook`), invoked by the `talk` controller — the dialogue-responder
contract. The spec still surfaces the tree to the controller, warms the
brain at wire time, and (via `BehavedMixin.getInstanceContributions`)
marks the host conversational for the `talk` affordance. See
[npc-dialogue.md](./npc-dialogue.md).

`addressed` and `given` are deferred (dialogue Wave 2 / later); both slot
in as additional `handleMessage` topic predicates with no new event.

## Ambient pacing budget

**These are starting points we expect to tune by feel — not a spec.** The
mechanism (below) is fixed; the *numbers* are provisional and dialable at
runtime. The goal is a consistent baseline as content grows so the world
doesn't drift toward chatter.

**The governing principle: responsive over ambient.** A room feels alive
because NPCs react to *you*, not because they talk to themselves. Those are
two separate budgets:

- **Responsive beats** — a brain firing on a **witness** trigger (you
  arrived, spoke, emoted) or a `talk`/`engage` open. Earned by player
  action, these are the good stuff; fire them freely. They do **not** count
  against the ambient budget.
- **Ambient idle** — a **cadence** brain firing unprompted ("signs of life
  when nothing's happening"). This is seasoning, and it's what grates when
  overdone. Keep it scarce. Silence is legitimate — a quiet room is often
  better than a busy one.

The budget (tune by feel; NPCs are rare hand-built carves, so a room is
usually 1–3 of them and per-NPC pacing sets the room aggregate):

| Scope | Ambient idle rate | Cadence |
|---|---|---|
| A whole room, all sources | ~1/min, ceiling ~2/min | — |
| A "normal" NPC | ~1 per 3–5 min | `cadence:180s`–`300s` |
| A deliberately chatty/performative NPC (rare) | ~1 per 1–2 min | `cadence:60s`–`120s` |
| A stoic / background NPC | 5–10 min, or **event-only** (no idle spec) | `cadence:300s`+ |
| Hard floor, anything unprompted | never under ~60s | (enforced, below) |

Two multipliers matter as much as the interval:

- **Variety beats frequency.** Even at 4-minute spacing, a 3-line pool
  reads as a loop. Give ambient pools generous entries; the sampler avoids
  immediate repeats.
- **Speech is heavier than emotes.** A wordless `{kind: free}` emote
  ("*paces the curb*") is cheap on attention; a `{kind: say}` line draws the
  eye like a real message. Bias idle pools toward emotes and let *speech* be
  mostly responsive.

**The global dial.** Two AppSettings retune the whole world's talkativeness
at once (playtest without editing every NPC), applied in
`BehavedMixin._effectiveCadence` before jitter:

- `behavior.ambientCadenceScale` (default `1.0`) — multiplies every
  *ambient* cadence interval. `2.0` = everything half as often. Authored
  cadences stay *relative* (a stoic NPC vs a barker); the dial moves them
  together.
- `behavior.ambientCadenceFloorMs` (default `60000`) — the anti-spam floor
  no ambient beat may fire faster than, whatever an author sets.

Both read per re-arm (a live `config` change lands on the next beat) and
default to identity (scale 1, floor 0) when app-settings is unwarmed, so
unit tests keep their fast cadences.

**Scope: the dial touches only ambient cadence.** A brain is ambient by
default; a **functional poller** whose timing is load-bearing sets
`static ambient = false` (today `shifts` reading roster state, `covers`
checking for an absent maker) so its authored interval is honored exactly.
Witness triggers are never scaled — responsiveness is never throttled.

## Slot contention

Concurrency rides the shared `EngagedMixin` slot map
(`body`/`hands`/`attention`/`voice`). A brain declares `claims` and
`requiresFree` in its descriptor (brain-declared, not author-set — the
spec stays `{ brain, trigger, config }`).

- A **cadence** brain **yields** a tick if any of its `requiresFree`
  slots is occupied in the host's engagement map.
- A **witness** brain that `claims` a slot holds it briefly via a
  **`BehaviorBeat`** — a short generic `DurativeActivity` that occupies
  the slot then completes with no effect. That makes the claim *real and
  time-extended* in the shared map.

The worked case: `wanders`/`patrols` claim `body`, require `attention`
free. When a player arrives, `greets` (witness) claims `attention` and
starts a beat → the wanderer's next cadence tick sees `attention`
occupied and yields ("the NPC stops wandering while it greets you") →
resumes on the next tick once the beat completes. The default is
**witness-preempts-cadence**, with no priority numbers; resume is
next-tick re-start, not a suspended-engagement resume.

## The `NPC` class

```ts
// lib/npc/NPC.ts
export class NPC extends BehavedMixin(PostRegistrationMixin(Character)) {}
```

Composition order is load-bearing. `clone` only invokes `postRegister`
on a host that composes `PostRegistrationMixin` (the marker), so `NPC`
must include it. `BehavedMixin` is **outermost** so the single
`postRegister` the clone pipeline calls resolves to *its* override
(which wires the behaviors); `PostRegistrationMixin` sits below to supply
the marker and the terminal no-op. (`CommandGiver`'s own `postRegister`
deeper in the chain is shadowed, but it self-seeds lazily, and NPCs emit
through Apis directly rather than the command system.)

`Behaved` is **branch-agnostic** — composable on any `Stuff`. Wave 1
puts it only on `NPC`, which keeps automated behavior **off player
Avatars** (they extend `ShelledCharacter`, not `NPC`) and off base
`Character`. Reactive scenery (a murmuring door, a bubbling fountain) is
the same mixin on a `Thing`/`Location` host later; live wiring (timers,
the seen-set) is runtime-only and re-installed from the persisted
`behaviors:` data on every clone/reboot.

## The canned brains

| Brain | Trigger | claims | requiresFree | Emits | config |
|---|---|---|---|---|---|
| `idles` | cadence | — | `voice,attention` | one sampled pool entry (emote / free / say) | `{ pool: {kind,value}[] }` |
| `random-chatter` | cadence | `voice` | `voice,attention` | a random spoken line | `{ lines: string[] }` |
| `wanders` | cadence | `body` | `attention` | traverse a random available exit | `{ avoid?: string[] }` |
| `patrols` | cadence | `body` | `attention` | traverse the next route direction (index in `state`) | `{ route: string[] }` |
| `greets` | `arrival` | `attention` | — | greet the arriver (directed) | `{ lines: string[] }` |
| `reacts` | `emote` | `attention` | — | emote/speak back at the perceived actor | `{ reactions: {to?,emote?,respond?}[] }` |
| `shifts` | cadence | — | — | migrate by employment shift state (teleport) | `{ behindBar, offstage, railStool? }` |
| `covers` | cadence | — | — | proprietor covers when no on-shift maker is present (`beginCover`/`endCover`) | `{}` |
| `tree-dialogue` | `engage` | `voice,attention` | — | none — reached via `open`, opens a `DialogueConversation` ([npc-dialogue.md](./npc-dialogue.md)) | the dialogue tree |
| `introduces` | `arrival` | `attention` | — | introduces the host to a newcomer (`learnIdentity`) unless already known | — |

(The trait-aware `converses` brain — cadence, claims `voice` — is documented
in [trait.md](./trait.md).) The speech/idle cadence brains declare
`requiresFree: [voice, attention]` so they fall quiet while the host is
mid-conversation (a `DialogueConversation` holds both slots) — the spoken
dialogue isn't muddied by ambient chatter.

`shifts` reads the host's shift state from the **employment engine**
(`EmploymentApi.shiftStateOf`, a sync read of the roster-maintained
`Employment.status`) and migrates the NPC to `behindBar` (on-shift) or
`offstage` (off-shift) — presence is now a *consequence* of employment
state, not a clock read (the schedule lives on the Business roster; see
[employment.md](./employment.md)). `railStool` is a reserved config key for
the deferred off-shift-at-the-rail presence. It is **not** presence-gated (it
must run unwatched to move off-stage cast). The sibling **`covers`** brain is
the proprietor's cover-driver: on a presence-gated cadence, if no other
active on-shift maker is present it `beginCover`s a transient unpaid
`MakerMixin`-conferring shift so an `order` still finds a fulfiller. This is
presence/migration only — the in-room shift-*change* ritual (count-out,
reconcile, hand-off) is a later scripting wave.

## Dev workflow & isolation (path-based)

The isolate → verify → broadcast loop falls out of the path-resolution
grain — the same code at two paths is two independent hot-reload registry
entries, so "which path the spec names" *is* the isolation boundary and
"which path you reload" *is* the broadcast scope.

- **Isolate**: a work-in-progress brain at a distinct path (e.g.
  `/domain/lounge/behavior/patrols-wip` — `/domain/` is an allowed
  prefix) wired onto a throwaway test NPC; editing + reloading it leaves
  live NPCs on the canonical path untouched.
- **Broadcast**: promote the WIP module to the canonical
  `/lib/behavior/patrols` and reload that path — every live NPC
  re-resolving it picks up the new code on its next fire. NPC specs name
  a stable path; publishing swaps the *code behind* the path, no per-NPC
  edits.

The polished lease-scoped **sandbox**, the **holodeck**, and the
drafts→staging→**publish gate** are deferred (the access +
authoring-intelligence builds; see [cms.md](./cms.md)). They drop in
later for free **because** brain resolution stays purely path-driven —
no site hardcodes a brain's canonical path, and no registry assumes one
path per brain-name.

## CMS save-gate validation

References are validated at two points: at resolution time (the path
loads or `act` is skipped) and at the **save-gate**.
`CmsLogic._writeContent` calls `validateBehaviorPaths(data)` after
parsing a content template — for every `behaviors[].brain` string it
runs `StuffApi.resolveExport(path, 'brain')` and rejects the save with
`CmsError('invalid')` if the brain doesn't resolve, so a typo'd or
dangling brain path is caught at author time, not silently at spawn.

## Downstream seams

- **Traits** (next build) read the host inside `act(ctx)` to bias
  behavior; emission channels can later carry a disposition valence on
  the same act-signature. `act` is the obvious read-point — not built
  yet.
- **npc-dialogue Wave 2**: `tree-dialogue` / `intent-dialogue` are
  brains in the same `behaviors:` model; the `addressed` trigger is
  another `handleMessage` topic predicate (the speech frame already
  arrives; dialogue adds the directed-`--to` discrimination). The
  `BrainContext.perceived` shape is general enough to host a responder.
- **Reactability gap**: NPC ambient speech/emote runs under a
  `ScheduleApi` `runRoot` frame, which carries no `commandId`, so those
  acts aren't registered as reactable and NPC→NPC `reacts` are silently
  skipped (player-originated acts have a `commandId` and resolve). Fine
  for Wave 1; revisit when NPC acts should themselves be reactable.

## The cast

| Element | Lives in | Role |
|---|---|---|
| `BehavedMixin` + `Behaved` | `lib/behavior/Behaved.ts` | Reads `behaviors:`, wires triggers, re-resolves brains, runs slot contention |
| `BehaviorSpec` / `BrainContext` / `BrainStatics` / `parseTrigger` vocab | `lib/behavior/brain.ts` | The brain category contract + trigger alias table |
| `BehaviorBeat` | `lib/behavior/BehaviorBeat.ts` | Generic short `DurativeActivity` that holds a slot for the contention window |
| The canned brains | `lib/behavior/{idles,random-chatter,wanders,patrols,greets,reacts,shifts,covers}.ts` | Path-resolved strategy modules (`covers` = the proprietor cover-driver; see [employment.md](./employment.md)) |
| `NPC` | `lib/npc/NPC.ts` | `Character` + `Behaved` archetype |
| `StuffApi.resolveExport` / `resolveExportSync` | `api/stuff.ts` | Path → fs → hot-reload registry brain-export seam |
| `validateBehaviorPaths` | `obj/api/CmsLogic.ts` | Save-gate brain-path validation |
| `Mixins.Behaved` | `lib/mixin.ts` | Mixin registry marker |

## Cross-references

- [activity.md](./activity.md) — the consumed substrate: `SchedulerApi`,
  `EngagedMixin` slots, `SustainedEngagement` / `DurativeActivity`,
  `ScheduledEmission`. Behavior is its first *behavior* consumer.
- [hot-reload.md](./hot-reload.md) — `HotReloadApi.getCurrentExport`, the
  class-like export retention, the re-resolve-per-use HMR pattern.
- [messaging.md](./messaging.md) / [emotes.md](./emotes.md) /
  [comms.md](./comms.md) — the emission channels (`say`/`whisper`/`shout`,
  `emote`/`emoteFree`) and the `SensorMixin.handleMessage` witness seam.
- [reactions.md](./reactions.md) — `ReactionApi.actInfo` speaker-recover
  used to resolve the subject of a witnessed emote/speech.
- [time.md](./time.md) — `WorldClockApi` + `DefaultCalendar`, the
  game-clock guard `shifts` reads.
- [cms.md](./cms.md) — the content surface NPC templates are authored in;
  the deferred holodeck / sandbox / publish gate.
- Seeding slate: npc-behavior-slate.md
  (Waves 2/3, the Traits build, and the LLM/scripted brain tails live
  here). The Wave 1 requirements + plan were retired into this doc at the
  pre-merge sweep.

## History

Wave 1 — the `Behaved` mixin, the seven canned brains, the path-resolved
brain category + `StuffApi.resolveExport`, the thin `NPC` class, the
CMS save-gate, and the Dave's Bar cast — landed on
`feature/npc-behavior-build` (MR !90). It is the first **behavior**
consumer of the activity substrate (respiration was the first engagement
producer). The requirements and plan docs were retired into this
subsystem doc at sweep time per the workflow's artifact-retirement rule.
