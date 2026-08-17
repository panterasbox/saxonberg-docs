# Senses

The multi-sense perception substrate. Content authors write rooms
once with `<sense channel="X">` regions and per-sense `Detail` slot
maps; viewers see only what their sensorium can perceive; the four
contact-family single-sense verbs (`smell` / `listen` / `feel` /
`taste`) and the gestalt verb (`sense`) all route through the same
substrate. Auto-on-entry fires `sense` so a player perceives a new
room across their full sensorium without typing.

This doc covers BOTH halves of the substrate:

1. **The authoring half** — `<sense channel="X">` MML wrappers,
   per-sense `Detail` slot maps, the `senseStripAugmenter`. Shipped
   in the 2026-06 senses build.
2. **The physics half** — `PerceptionApi`, the seven `Modality`
   singletons (vision / smell / sound / touch / taste / verbal-esp
   / emotive-esp), the field propagation walks for vision / smell /
   sound, contact thermoreception for touch, per-frame modality
   attribution at `Scene.modality` + `SensorMixin.filterMessage`,
   organ-gates-modality with the augmentation widening
   (`PerceptionApi.sensorium` walks BodyPlan organs + active-mixin
   `_grantsModalities`). Shipped in the 2026-06 perception build.

The slate's deferred items — smell trails / temporal persistence,
active-sense (echolocation), full ESP field walk, per-species
hearing / tactile / gustatory profiles — remain Wave 3+. (Vitals
burn-damage on scalding contact, listed deferred here originally,
shipped with the thermal build — see [thermal.md](./thermal.md).)

## Substrate consumed

- [light.md](./light.md) — vision substrate; unchanged. `look` stays
  vision-only (now expressed via the augmenter's `filter: ['vision']`).
- [race.md](./race.md) — `BodyPlan` + `Species`. This build adds
  `BodyPlan.getModalities()` and `Species.olfactoryProfile`.
- [messaging.md](./messaging.md) — the `MarkupAugmenter` pipeline.
  The new `senseStripAugmenter` slots onto `VisibleMixin.markupAugmenters`.
- [message-rendering.md](./message-rendering.md) — MML vocabulary.
  The `<sense channel="X">` tag is a new semantic core member; flatten
  + parse already handle unknown tags transparently, so the wrapping
  ships without renderer / client changes.
- [persistence.md](./persistence.md) — `Detail` rides the
  instruction-field path (`applyDetails` Phase 2);
  `Species.olfactoryProfile` rides the scalar-default rule the same
  way `visionProfile` does.

## What ships in this build

### `SenseChannel` vocabulary

Canonical physical-sense channel union, declared in
`lib/description/Perceiver.ts` alongside the runtime
`SENSE_CHANNELS` array:

```ts
export type SenseChannel = 'vision' | 'hearing' | 'smell' | 'touch' | 'taste';
export const SENSE_CHANNELS: readonly SenseChannel[] = [...];
```

Perceiver is the actor-side surface that perceives across these
channels, so the vocabulary's home is here — BodyPlan declares
which ports a body has but uses this type to label them; Detail
stores the prose per channel but uses this type to key its slots.

Used consistently across every surface: `SensoryPort.modality`,
`Detail` per-sense slot map keys, `<sense channel="X">` MML wrapper
attribute, `senseStripAugmenter` filter, the four `requires*` verb
validators. The eyes-modality entry uses `'vision'` (the
channel/process word), NOT `'sight'` (the organ word). ESP / alien
channels are deliberately NOT in this union for v1 (slate Wave 3).

### Per-sense `Detail` slot map

`DetailedMixin`'s `Detail` shape grew from a single `description: string`
to a per-sense slot map. Authoring accepts both shapes:

```yaml
# New per-sense authoring.
details:
  bookcase:
    keywords: [shelves]
    vision: "Hand-tooled leather spines."
    touch: "Smooth walnut, grain runs vertical."

# Legacy authoring — still works, populates the `vision` slot.
details:
  bookcase:
    keywords: [shelves]
    description: "A tall walnut bookcase."
```

Mixing the two (`{ description: "...", vision: "..." }` in the same
entry) is rejected at applier time with a clear error — pick one
shape per entry.

Lookup gained an optional `sense` argument:

```ts
host.getDetail('bookcase');           // → vision slot (back-compat)
host.getDetail('bookcase', 'vision'); // explicit, same as above
host.getDetail('bookcase', 'touch');  // touch slot, or null
host.getDetail('bookcase', 'touch', 'parent'); // sense + nested parent
host.getDetail('lock', 'handle');     // legacy parent-arg shape preserved
```

The 2-arg dispatch distinguishes sense-vs-parent by recognising the
five `SenseChannel` literals — any other string is treated as the
legacy parent path.

`setDetail` gained an overload accepting a slot map:

```ts
host.setDetail(['bookcase'], 'A tall walnut bookcase.');                 // legacy
host.setDetail(['bookcase'], { vision: "...", touch: "..." });           // new
```

Per-field invariants on the slot-map shape: at least one slot must
be populated; every populated slot must be a string. The wire
projection (`getDetailEntries` / `getDetailEntry` consumed by the
inspection-card subscription substrate) projects the `vision` slot
into the existing `description` field — non-vision slots are
server-side state only; the v1 wire stays single-channel for back-compat.
Persistence migration is implicit — pre-existing documents with
`{ description: "X" }` re-hydrate transparently into the `vision`
slot on first load.

### `<sense channel="X">` MML wrapper

A new semantic MML tag parallel to `<chan>` / `<player>` / `<mention>`.
Lives in any long description; the server-side `senseStripAugmenter`
drops regions the viewer can't perceive before the body ships to the
client. The flatten / strip-tags failsafe emits children verbatim
(`Mml.flatten('<sense channel="smell">garlic</sense>')` = `'garlic'`),
so the wrapping adds no failsafe artifacts. The client's `parseMml`
recognises it as a tag node with `channel` attr without any client
changes — unknown tags already round-trip transparently.

`<detail key="X" sense="Y">` adds an optional `sense=` attribute to
the existing `<detail>` tag. Default-absent = `'vision'` (the
back-compat default; existing detail authoring keeps rendering
unchanged).

### `Mml.stripBySense`

The strip primitive lives on `Mml` (per the hard rule that nothing
outside `api/mml.ts` may import `api/mml/`). Two rules:

| Tag                                | Channel ∈ allowed | Channel ∉ allowed             |
| ---------------------------------- | ----------------- | ----------------------------- |
| `<sense channel="X">…</sense>`     | keep tag + recurse | drop tag AND children       |
| `<detail key="K" sense="X">…</detail>` | keep tag + recurse | drop tag, KEEP children inline |
| All other tags                     | preserve, recurse | preserve, recurse             |

Untagged prose is always preserved.

### `senseStripAugmenter`

A new entry on `VisibleMixin.markupAugmenters`. Reads the per-call
`opts.filter` (a `readonly SenseChannel[]`) and the viewer's
sensorium (from `PerceptionApi.sensorium(viewer)`); strips
regions whose channel isn't in `filter ∩ sensorium`.

```ts
function senseStripAugmenter(text, host, viewer, opts?) {
  const sensorium = PerceptionApi.sensorium(viewer);
  const filter = opts?.filter ?? sensorium;          // gestalt fallback
  const allowed = new Set(filter.filter((ch) => sensorium.includes(ch)));
  return Mml.stripBySense(text, allowed);
}
```

The augmenter ordering: `senseStripAugmenter` runs FIRST
(parent-first walker, Visible above Detailed in the typical chain),
`wrapDetailKeysAugmenter` runs SECOND. Strip-then-wrap is correct
because wrapping inside a region destined for the strip is wasted
work.

### `Mml.augment` + `MarkupAugmenter` widened with `opts?: AugmentOpts`

The substrate walker is exposed as `Mml.augment(text, host, viewer,
opts?)` — a static on the `Mml` class. Callers thread through the
class handle: `import { Mml } from '../../api/mml'; Mml.augment(...)`.

The augmenter contract is `(text, host, viewer, opts?: AugmentOpts)
=> string`. The optional `opts` parameter is the senses build's
addition; older 3-arg augmenters keep working (TypeScript covariant
params). Verbs that care about the filter pass it explicitly:

```ts
location.getMarkupLong(viewer, { filter: ['vision'] });             // look
location.getMarkupLong(viewer, { filter: ['smell'] });              // smell
location.getMarkupLong(viewer, { filter: PerceptionApi.sensorium(viewer) }); // sense
location.getMarkupLong(viewer);                                     // subscription — gestalt default
```

The inspection-card subscription's `read = (stuff, viewer) =>
stuff.getMarkupLong(viewer)` passes no opts and naturally gets the
viewer's full sensorium — the right "what does this viewer perceive
right now?" projection for the cockpit.

### `BodyPlan.getModalities()` + `PerceptionApi.sensorium(viewer)`

`BodyPlan.getModalities(): SenseChannel[]` returns the deduped
channel list from `sensoryPorts`. A sessile body plan with no ports
returns `[]`.

`PerceptionApi.sensorium(viewer)` is the canonical viewer →
sensorium walker. Walks viewer → Organism → Species → BodyPlan →
`getModalities()`; returns `[]` when any step is null (a
non-Organism viewer, an Organism without a Species, etc.). Shared
by the augmenter AND the four `requires*` validators.

`SenseChannel` is declared in `lib/description/Perceiver.ts` (the
actor-side perception surface). `PerceptionApi.sensorium` lives
on the API class so consumers thread through the api/ layer rather
than importing a bare function from lib/.

### `Species.olfactoryProfile`

New persistent field on `Species`, parallel to `visionProfile`. v1
ships a single coarse `acuity` scalar:

```ts
export interface OlfactoryProfile {
  acuity: 'keen' | 'normal' | 'dull' | 'none';
}
```

No propagation walk means no math consumes the value — the field's
job in this build is shape + place for content authors to start
declaring per-species smell variance. When the
`PerceptionChannel` substrate lands and smell gains a propagation
walk, this extracts to a `lib/perception/Smell.ts` value-object
module following the `Light.ts` precedent.

No `hearingProfile` / `tactileProfile` / `gustatoryProfile` yet —
those land per content demand.

### Single-sense verbs: `smell` / `listen` / `feel` / `taste`

Four contact-only verbs sharing a common `SingleSenseControllerBase`.
Each subclass pins two abstract properties: `senseChannel` (the
`SenseChannel` literal — note `listen`'s channel is `'hearing'`,
not `'listen'`, because channel ≠ verb name) and `sceneTopic` (the
dotted Scene topic). Bare form renders the current location filtered
to that channel; targeted form resolves via MQL and reads
`host.getDetail(dotted, senseChannel)`.

| Verb     | Channel   | Topic                              | Validator         |
| -------- | --------- | ---------------------------------- | ----------------- |
| `smell`  | `smell`   | `sense.survey.smell`     | `requiresSmell`   |
| `listen` | `hearing` | `sense.survey.listen`    | `requiresHearing` |
| `feel`   | `touch`   | `sense.survey.feel`      | `requiresTouch`   |
| `taste`  | `taste`   | `sense.survey.taste`     | `requiresTaste`   |

Topics are **verb-named, organized hierarchically under
`sense.survey.*`**. Topic = the kind of event the frame
represents; channel attribution lives separately in body MML. See
the *Perception topic vocabulary* section below for the full
organizing principle.

Verb-level validators in `lib/command/validators/requires*.ts` gate
the giver-side sensorium check via
`PerceptionApi.canPerceive(giver, PerceptionApi.modalityByName(channel))`.
Same pattern as `requiresAnimate`. Polite refusal strings: `"You
can't hear."` / `"You have no sense of smell."` / `"You can't feel
anything."` / `"You have no sense of taste."` Failure routes through
the dispatcher's standard validator-failed prose path.

A non-Detailed target gets a polite "you don't perceive anything
notable" response. Targets outside the viewer's MQL scope aren't
addressable — contact-only is enforced by the existing scope rules,
no extra code.

No aliases (`sniff` / `lick`) in v1 — keep the surface tight.
`--peek` not yet supported on the single-sense verbs.

### Gestalt verb: `sense`

The dominant room-presentation verb post-this-build. Bare form
renders the current location with the augmenter's filter set to the
viewer's full `getModalities()`; targeted form mirrors `look <target>`'s
shape with the gestalt filter. Detail lookup uses `'vision'` (per
the slate's "click = look" rule — the gestalt's detail-drill default
stays single-sense; smell/touch/etc. require the verb-specific
single-sense form).

`SenseController` keeps the room-presentation chrome (exits,
occupant list) that `LookController` had — the vision-bound
affordances stay because `sense` IS the room-presentation verb
now.

**Topic is fixed: `sense.survey.sense`.** Topic =
kind-of-event (the gestalt act). Channel attribution rides body
MML — the viewer-visible body contains `<sense channel="X">`
regions for whichever channels survived the filter intersection.
Filter-by-channel in the cockpit is a body-MML concern, not a
topic concern.

No `requiresVision` / `requiresSense` validator — the gestalt filter
is the viewer's full sensorium; the augmenter naturally produces an
appropriate render for whatever channels the viewer has (vision-only
authoring renders identically for a vision-bearing viewer; a
sightless viewer's gestalt strips the vision regions and presents
the rest).

### Perception topic vocabulary

Topics across the system follow the principle: **topic = the kind
of event/data the frame represents**. Hierarchical, with wildcards
at any level. Per-verb leaves at the bottom; kind-level
intermediates group leaves for wildcard aggregation. Channel
attribution is orthogonal — it lives in body MML, not topics.

The perception family looks like this:

```
sense.
├── sense.       ← verbs (deliberate query)
│   ├── look      LookController
│   ├── sense     SenseController
│   ├── smell     SmellController
│   ├── listen    ListenController
│   ├── feel      FeelController
│   ├── taste     TasteController
│   └── scry      ScryController
├── ambient.     ← channels (passive arrival, no verb)
│   ├── vision
│   ├── hearing
│   ├── smell
│   ├── touch
│   └── taste
├── measurement.    ← instrument-mediated quantitative readouts
│   ├── measure-temperature      MeasureTemperatureController
│   ├── measure-pressure         MeasurePressureController
│   ├── measure-humidity         MeasureHumidityController
│   ├── measure-gravity          MeasureGravityController
│   ├── measure-altitude         MeasureAltitudeController
│   ├── measure-light            MeasureLightController
│   ├── measure-atmosphere       MeasureAtmosphereController
│   ├── analyze-atmosphere       AnalyzeAtmosphereController
│   ├── analyze-chemistry        AnalyzeChemistryController
│   ├── analyze-light            AnalyzeLightController
│   └── weigh                    WeighController
├── search.      ← search/match results
│   ├── find      FindController
│   └── locate    LocateController
└── inventory    Get/Drop/InventoryController
```

Each sub-family has its own axis:

- **sense** is verb-leafed. Each verb is a distinguishable kind of
  deliberate query producing a different shape of prose.
- **ambient** is channel-leafed because ambient events have no verb
  to name them — they're "the world fed perceptual content
  unbidden, on this channel." Five channel leaves mirror the
  `SenseChannel` vocabulary.
- **measurement** is verb-leafed (per-instrument). Each instrument
  reads a different physical quantity; the leaves are fine-grained
  so per-instrument styling/filtering works.
- **search** is verb-leafed. find / locate are different scopes of
  search (local vs world-scope).

Wildcards aggregate: `sense.survey.*` matches all
deliberate first-person perception; `sense.ambient.*`
matches all unbidden perceptual input; `sense.reading.*`
matches all instrument output.

Shell-query verbs land outside the perception family — they're
state queries about the player's own setup, not perceptual events:

```
shell.result.
├── alias       AliasController
├── var         VarController
├── settings    SettingsController
├── focus       FocusController
└── player      PlayerController
```

(Plus the pre-existing `shell.result.{author, fs, help, movement}`
leaves.)

**Other agents:** an NPC speaking rides `speech.vocal` exactly
like a player speaking — different *generators*, same kind of
event. The agent's identity rides payload/actor fields, not the
topic. Same principle for NPC emotes (`act.emote`), NPC actions
narrated to witnesses (`act.deed`), and NPC
perception (the NPC's own controller fires perception frames to
the NPC's own Sensor at the same perception topics).

### Auto-on-entry switches to `sense`

`MobileMixin.autoLookOnArrival` was renamed to `autoSenseOnArrival`
and its body now `forceCommand`s `sense` (not `look`). Same focus-reset
behaviour, same forceCommand plumbing, same error-swallow. The
four call sites that forced `look` on arrival all delegate through
this single hook:

- `Avatar.enter` (post-welcome auto-render)
- `Mobile.traverse` (after a movement)
- `Mobile.teleport` (after an instant move, when not silent)
- `Goto -l` flag fallback (raw-move path)

Existing rooms render identically because the augmenter's
filter ∩ sensorium = the viewer's full sensorium for a room with
no `<sense>` regions authored — vision-bearing players see the same
vision-only prose they did before. The door is open for multi-sense
authoring without a breaking transition.

Bare `look` becomes opt-in vision-only — players who want JUST the
vision channel still have the verb; the gestalt is the default.

## Authoring discipline

The slate's "events single-channel per frame, state multi-sense"
discipline:

- **Events** — `Scene.send` carries one frame per audience per
  channel. A frame attributes one channel. Cross-channel events
  (a spell that glows AND hums) are two separate sends, one per
  channel. Don't conflate channels at emit time.
- **State** — the room's long description and every Detail's per-sense
  slot map are multi-channel together. One `<sense channel="X">`
  region per channel per location; one slot per channel per Detail.
  Authors describe the thing across senses in one place; the
  augmenter filters per-viewer per-verb at render time. No
  "Smell: nothing. Sound: nothing." rule — empty channels just don't
  appear.

This split means content authors don't write the same thing five
times; the substrate's filter does the per-viewer per-verb work.

## The physics substrate (2026-06 perception build)

### `Modality` singletons + `PerceptionApi`

The physics half centers on a single `PerceptionApi` and a
`Modality` base class modeled on
[`LocomotionMode`](./locomotion.md):

```ts
// lib/perception/Modality.ts
export class Modality extends SingletonMixin(PropertiedMixin(Idea)) {
  // Persistent: name, family ('field' | 'contact' | 'network'),
  // modality (BodyPlan organ key).
  public signalAt(loc): Signal | null { return null; }
  public perceiveFor(viewer, loc, signal): Percept | null { return null; }
}
```

Seven concrete subclasses under `lib/perception/modalities/`:
`VisionModality`, `SmellModality`, `SoundModality`, `TouchModality`,
`TasteModality`, `VerbalESPModality`, `EmotiveESPModality`. Each
has a seed YAML at `seeds/lib/perception/modalities/<name>.yaml`;
`bootstrap.ts`'s `templatePathPrefix` entry clones the seven at
boot.

`PerceptionApi` (the single Api):

```ts
class PerceptionApi {
  static modalityByName(name: string): Modality;
  static modalityByOrganKey(key: string): Modality | null;
  static signalAt(loc, modality): Signal | null;
  static perceiveAt(viewer, loc, modality): Percept | null;
  static sensorium(viewer): readonly Modality[];
  static canPerceive(viewer, modality): boolean;
}
```

Modality singletons are addressed by **name** in `modalityByName`
(`'vision'`, `'smell'`, `'sound'`, `'touch'`, `'taste'`,
`'verbal-esp'`, `'emotive-esp'`) and by **organ key** in
`modalityByOrganKey` (`'hearing'` for the sound modality;
`'verbal-esp'` for the verbal-ESP modality). For physical modalities
the two diverge only on sound (name 'sound', organ key 'hearing').

### Propagation walks (field modalities)

Vision, smell, and sound implement their own propagation walk in
`Modality.signalAt`. The walks share structural shape (ambient +
contents-side emitters + fixture-side emitters + cross-boundary via
Conduit + cross-exit at base transmissivity, depth-capped via
`MAX_HOPS = 2`, cycle-guarded via a `visited` set, vacuum scope
blocks at the recipient) but remain independent code per the
"per-modality walks, not a generic walker" decision in the
requirements doc.

- **Vision** (relocated from the retired `LightApi.lightAt`):
  lumen-flux accumulator, sources → `Light` value object with lux
  intensity + flux-weighted color temperature + capped 3-entry
  source list. Door + Window participate via `LightConduit`.
- **Smell**: ppm concentration accumulator with per-identity
  totals, dominant identity by total concentration, capped 3-entry
  source list. Door + Window participate via `SmellConduit`.
- **Sound**: linear-amplitude accumulator (`10^(dB/10)` terms)
  converted back to dB at wrap — physically correct merge for
  incoherent sources. Two 60 dB sources sum to ~63 dB (not 120).
  Door + Window participate via `SoundConduit`. Ambient floor seeds
  from the universe-root biome's `_defaultAmbientSoundLevel` (full
  async biome-chain integration deferred).

### Touch (contact modality)

Touch is contact-only — no propagation walk. `TouchModality` exposes
two surfaces:

- `signalAt(loc)` (sync substrate contract) reads only the scope's
  inline `_temperature` field.
- `touchAt(loc, detailKey?)` (async) walks the full biome chain via
  `BiomeApi.resolveTemperatureFor`. `FeelController`'s bare-form
  and detail-via branches consume this.

`Touch` carries a K temperature plus a coarse band (`cold` / `cool`
/ `comfortable` / `warm` / `hot` / `scalding`) calibrated against
the universe-baseline 295 K. Bare `feel` prepends "The air feels
<band>."; detail-via `feel <target>` prepends "It feels <band>."
above the per-Detail `touch` slot prose.

The thermal build connected the contact senses to the heat-exchange
layer: `feel <object>` on a Thermal object now reports the object's
own **surface** temperature band (a sealed insulated vessel reads
~ambient though its contents scald — the surface-vs-contents sensory
gate), and the previously-deferred burn hook went live —
`Touch.contactBurn` afflicts a burn trauma on scalding-band (≥ 345 K)
contact via `feel` and a bare-handed `get`. See
[thermal.md](./thermal.md).

### Per-frame modality attribution

`MessageFrame.meta` gains `modality?: string`. Producers stamp it
via `Scene.modality(name)` at compose time:

- `VocalMixin.say` → `'hearing'`
- `AetherMixin.tell` → `'verbal-esp'`
- Multi-modality events (door slam = sight + sound) stay authored
  as separate `Scene.send` calls per modality, one frame per
  modality.

System / log / narrative frames omit the call and deliver
unconditionally.

`SensorMixin.filterMessage` consults the recipient's
`PerceptionApi.sensorium` (organ-key indexed) and drops frames
whose modality isn't in it. **Actor self-frames** (`audience:actor`
tag, auto-attached by `Scene.toSelf`) ALWAYS deliver — you always
perceive your own acts diegetically.

The augmenter's existing per-region `<sense channel="X">` strip is
**orthogonal** to this check. Both live: `filterMessage` decides
whether the frame arrives at all; the augmenter decides which
regions of the frame's body MML the viewer sees within an arrived
frame.

### Organ-gates-modality widened with augments

`PerceptionApi.sensorium(viewer)` walks (a) the viewer's BodyPlan
`sensoryPorts.modality` strings via `modalityByOrganKey`, AND (b)
`MixinApi.getActiveMixins(viewer)` collecting each active mixin's
`_grantsModalities`. Returns the deduped union.

Augment conferral flows in transparently — the augment doesn't
enumerate modalities itself; `MixinApi.getActiveMixins` includes
augment-conferred mixins automatically, and those mixins' grants
flow through. v1 case: every Avatar bootstraps a
`AetherImplant` that confers `AetherMixin`; AetherMixin's
`_grantsModalities = ['verbal-esp', 'emotive-esp']` then surfaces
in the sensorium. See [augmentation.md](./augmentation.md) for the
full substrate.

`SpeciesApi.deriveSensorium` retired; callers route through
`PerceptionApi.sensorium`.

### Hybrid ESP framing

ESP modalities are tagged `family: 'field'` honestly — the
underlying physics IS a field (the aether). v1 ships only the
network-routing layer (`AetherMixin.tell`, renamed from the
verb-level `tell` to `dm`); the field-half (eavesdropping in range,
encryption stripping for non-addressee dms, the local aether-field
walk via a future `AetherConduit`) is reserved for a future
ESP-physics build. `VerbalESPModality.signalAt` /
`EmotiveESPModality.signalAt` return `null` in v1 — the documented
reserved seam.

ESP organs are NOT declared on any v1 species' BodyPlan. ESP
arrives via the baseline comm implant (augment-conferred
AetherMixin). The fantasy/alien-species path stays reserved — a
future telepath species (magical empath, alien biological aether
receiver) can either declare an ESP modality on its BodyPlan via
the existing `sensoryPorts` mechanism, OR carry a slotted
biological-empathy organ that confers AetherMixin the same way the
implant does.

### Bare-verb upgrades

The four contact-family verbs upgrade their bare forms:

- Bare `smell` reads `SmellModality.smellAt`, applies the viewer's
  `Species.olfactoryProfile.acuity` threshold (keen:1 / normal:10
  / dull:100 / none:Infinity ppm), renders prose with source
  attribution.
- Bare `listen` reads `SoundModality.soundAt`, gates on a default
  10 dB hearing threshold, renders prose with source attribution.
- Bare `feel` prepends an ambient-temperature band line; targeted
  `feel <target>` with detailPath prepends per-detail temperature.
- Bare `taste` unchanged (contact-only authoring this build).

The targeted form of each verb keeps its per-Detail slot read
inherited from `SingleSenseControllerBase`.

### Masking

A viewer's effective threshold on a modality is
`max(viewer_threshold, ambient_signal_strength_on_modality)`. For
sound: ambient comes from the biome's `_defaultAmbientSoundLevel`
resolved at the viewer's scope. For smell: ambient comes from the
room's own smell signal (non-target sources). Per-frequency-band
masking and per-odor-class masking are out of scope.

### File layout (physics half)

```
lib/perception/
├── Modality.ts                    Substrate base
├── modalities/
│   ├── VisionModality.ts          Walk + static helpers (lightAt, bandAt, perceivedBand, canSee, shadowsAt)
│   ├── SmellModality.ts           Walk + smellAt static
│   ├── SoundModality.ts           Walk + soundAt static (dB arithmetic)
│   ├── TouchModality.ts           signalAt (sync inline) + touchAt (async biome chain)
│   ├── TasteModality.ts           (taste is contact authoring only)
│   ├── VerbalESPModality.ts       (signalAt null v1)
│   └── EmotiveESPModality.ts      (signalAt null v1)
├── Light.ts                       Value object + bandFor + LIGHT_BANDS + applyBandShift + compareBand + ShadowQuality
├── Smell.ts                       Value object (ppm + identity + sources)
├── Sound.ts                       Value object (dB + character + sources)
├── Touch.ts                       Value object (K + TouchBand) + bandFor
├── LightSource.ts                 Mixin + LightSourceObserver hook contract
├── SmellSource.ts                 Mixin + SmellSourceObserver hook contract
├── SoundSource.ts                 Mixin + SoundSourceObserver hook contract
└── (Perception.ts, AmbientLit.ts, Scryable.ts — substrate unchanged)

api/perception.ts                  PerceptionApi (single dispatch surface)
lib/boundary/SmellConduit.ts       Conduit interface
lib/boundary/SoundConduit.ts       Conduit interface
seeds/lib/perception/modalities/   Seven seed YAMLs
```

## What's NOT in this build (Wave 2+)

- **`PerceptionChannel` substrate abstraction.** No general
  five-part Channel object yet (emission / propagation+medium /
  attenuation+masking / sensitivity / rendering). The five physical
  senses route through specific code paths.
- **Field / contact / network family physics.** No propagation
  walks, no attenuation/masking, no field/contact/network family
  modelling. Smell is contact-only (no cross-room diffusion, no
  gradient, no falloff). Same for the other senses.
- **ESP-as-channel registration.** ESP channels (verbal, emotive)
  don't appear in `<sense>` MML wrappers in state authoring —
  `<sense>` is for physical senses only. The slate's "messaging =
  sensing" unification is deferred. Existing comms (`VocalMixin.say`,
  `AetherMixin.tell`) ship unchanged.
- **Smell trails / temporal persistence.** Slate Wave 3.
- **Light / vision convergence onto the new substrate.** `VisionModality`,
  `canSee`, `visionProfile` ship unchanged. `LookController` doesn't
  gain a `requiresVision` validator. The dark-room test fixture
  uses a sightless-by-construction sensorium; real-world
  darkness-blocks-vision lands when light converges onto the
  substrate.
- **Salience-threshold engine.** Authors decide what's notable by
  what they wrap in `<sense>`. Empty channels just don't appear; no
  engine threshold.
- **Sensorium-relative stealth.** Slate Wave 3.
- **Alien sense channels** (`echolocation`, `electroreception`,
  `pit-sensing`). Slate Wave 3 — proves the abstraction once the
  full `PerceptionChannel` substrate exists. Body plans declaring
  alien sensory ports don't yet unlock anything beyond appearing in
  `getModalities()`.
- **Per-sense Species profiles beyond `olfactoryProfile`.** No
  `hearingProfile` / `tactileProfile` / `gustatoryProfile` yet.
- **Vitals organ-condition modulation.** Slate Wave 2.
- **Per-channel instruments tie.** No instruments-emit-as-sense yet.
- **Inspection-card wiring changes.** Card consumes
  `getMarkupLong(viewer)` today; that path still works because the
  augmenter walk is per-viewer (default-absent opts → full sensorium).
- **Active-sense / emit-and-perceive-the-return pattern.** Slate Wave 3.
- **Single-sense verb aliases** (`sniff` / `lick`) and `--peek`.
  Land per content demand.
- **No client changes.** The `<sense>` MML tag and `<detail sense=>`
  attribute ride the existing client renderer transparently; the
  server-side augmenter strip means the wire body the client
  receives is already filtered per-viewer.

## Cross-references

- docs/slates/tails/senses-slate.md — the
  seeding slate. Names the wider design space; this build's surface
  is the cross-wave slice.
- [docs/subsystems/light.md](./light.md) — the vision substrate,
  unchanged by this build.
- [docs/subsystems/race.md](./race.md) — BodyPlan / Species. This
  build extends `BodyPlan.getModalities()` + adds
  `Species.olfactoryProfile`.
- [docs/subsystems/messaging.md](./messaging.md) — `MarkupAugmenter`
  pipeline + `Scene` builder.
- [docs/subsystems/message-rendering.md](./message-rendering.md) —
  MML vocabulary + the substrate that consumes augmenter output.
- [docs/subsystems/persistence.md](./persistence.md) — `Detail`'s
  instruction-field applier + the `Species.olfactoryProfile`
  scalar-default round-trip.
- [docs/subsystems/command-spec.md](./command-spec.md) — author
  guide for the verb framework the new verbs follow.

## History

Initial substrate ship (2026-06, MR !37). The design landed in
three MR-iteration rounds before the final shape; key shifts:

- **`SenseChannel` vocabulary home.** First proposed on
  `lib/species/BodyPlan.ts` (alongside `SensoryPort.modality`).
  MR feedback relocated it to `lib/description/Perceiver.ts` —
  the actor-side perception surface is the canonical home for the
  channel vocabulary; BodyPlan references the type but Perceiver
  declares it.
- **`Mml.augment` static (was bare `augmentMarkup` function).**
  Round 1 followed the existing pattern (bare function export
  from `api/mml.ts`); MR feedback folded it into the `Mml` class
  as a static method to thread through the API class handle.
- **`PerceptionApi.sensorium` (was a bare function on BodyPlan, then
  briefly `SpeciesApi.deriveSensorium`).** Same principle as augment —
  moved off the lib module onto an API class; the access/perception
  build settled it on `PerceptionApi`.
- **Topic vocabulary evolution.** Three rounds:
  - Round 1: per-verb topics (`sense.{look, smell,
    listen, feel, taste, sense}`).
  - Round 2: channel-named topics (`.vision`, `.hearing`, etc.),
    with `SenseController` deriving topic from filtered body
    content via the now-deleted `Mml.collectSenseChannels`.
    Reverted: collapsed too much (80% of frames at 6 topics) and
    conflated kind-of-frame with sensory-channel axes.
  - Round 3 (final): hierarchical verb-leafed topics under
    `sense.{sense, ambient, measurement, search}`
    parents. Channel attribution stays orthogonal — lives in
    body MML. Shell-query verbs (alias, var, settings, focus,
    player) moved to `shell.result.*` family. 22 controllers
    updated; ambient subtree uses channel-named leaves because
    ambient has no verb to name them.
