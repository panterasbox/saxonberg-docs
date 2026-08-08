# Character Generation (Intake)

How a brand-new player gets a body and a name. Covers the post-login
branch (roster vs. char-gen), the `enroll` verb as a draft state
machine, `Login` as the accumulator, the commit/spawn handoff, the
species/name-bank substrate, and the cockpit's char-gen phase.

This doc is the **intake** map. Sibling docs cover adjacent ground
without overlap:

- [connection.md](./connection.md) — the auth → WebSocket → `Interactive`
  → `Login` plumbing this rides on, and the disconnect/logout path.
  The Cast table (`User`/`Avatar`/`Interactive`/`Login`) lives there.
- [command-routing.md](./command-routing.md) — the MVC dispatch pipeline
  `enroll` is a verb on. char-gen adds no new dispatch mechanism.
- [race.md](./race.md) — `Species`/`BodyPlan`/`Material` substrate the
  species picks resolve against; char-gen consumes it, doesn't define it.
- [state-model.md](./state-model.md) — why there is no `Player` class,
  the `User.playerIds` ownership list, the template-vs-runtime split.

## The principle

**Char-gen rides the real command pipeline.** It is not a bespoke
wizard, not the prompt stack, not a special parser. `Login` is a
`CommandGiver`; `enroll` is an ordinary MVC verb dispatched through
`executeCommand`. Every pick is a real command (`enroll species human`)
that echoes in the command bar and is equally typeable from a bare text
client. The dedicated cockpit layout is a *skin* over that command
stream, not a parallel path.

**The server owns the draft; the client owns the layout.** The server
holds an unordered `EnrollmentDraft` and reports which fields are still
`missing` — it has no notion of a "current step" or cursor. The client
decides how to chunk the fields into screens. This split is deliberate:
a single-page variant of char-gen (for A/B testing) is a pure client
change, touching no server code.

## The post-login branch

When a connection authenticates, `Login` takes the `Interactive` and
branches on the user's character count (`Login.enter`,
`Login.ts:116`):

- **0 characters** → char-gen. `Login` runs the `enroll` flow itself
  (it's a real `CommandGiver`), accumulating picks in an
  `EnrollmentDraft` until `enroll confirm` commits a fresh `Avatar`.
  `Login` then destructs.
- **≥1 character** → the character-select roster. `Login` emits a
  `session.identity.roster` frame and stays alive, awaiting a
  `play <playerId>` (handled by `PlayController` → `Login.playCharacter`)
  or an `enroll` to mint an additional character. Multichar is on:
  signup creates **zero** avatars, so char-gen is the only path to a
  first character, and the roster is the hub thereafter.

Signup no longer provisions a default Avatar template — an abandoned or
disconnected char-gen leaves no playable character and no orphaned
template, because nothing persists until the commit's atomicity boundary
(below).

## `Login`: CommandGiver + Sensor + accumulator

```
const LoginBase = CommandGiverMixin(SensorMixin(HasInteractiveMixin(Idea)));
```

`Login` (`Login.ts:80`) is three things at once:

- A **`CommandGiver`** with a tight verb allowlist —
  `self: ['charactergen/enroll.yaml', 'charactergen/play.yaml']`
  (`Login.ts:90`). No world verbs leak into char-gen.
- A **`Sensor`**, so engine/narrative frames (welcome prose, enrollment
  narration, the char-gen state payload) reach the bodiless `Login` and
  render in the client. Sensory (modality-tagged) frames do **not**
  reach it — `Login` has no body, so smell/sound/sight channels have no
  receiver. This is what lets the same `MessageApi.scene(...).toSelf(...)`
  machinery serve a player who isn't yet embodied.
- The **home for the in-progress `EnrollmentDraft`** — a transient,
  never-persisted scratch object (`Login.ts:97`,
  `getEnrollmentDraft`/`setEnrollmentDraft`).

A locationless giver **dispatches** rather than being silently dropped:
the dispatch location guard was relaxed game-wide so `Login` (which is
intentionally placeless) can run verbs. Verbs that read location degrade
to an empty result rather than being gated; the embodied-avatar path is
unchanged. See [command-routing.md](./command-routing.md).

### `EnrollmentDraft`

The accumulator (`Login.ts:52`) carries the picks plus presentation
scratch: `speciesKey`/`speciesPath`/`speciesCommonName`, `sex`, `name`,
`surname`, `pronouns`, `aspiration`, the current name `suggestion`, the
caller's `realName`/`accountName` (from `resolveNames` — the OAuth/Twitch
display name, surfaced so the client can show "signed in as …"). It is
pure data; all validation and mutation lives in the controller.

## The `enroll` verb: a draft state machine

`EnrollController` (`obj/command/charactergen/EnrollController.ts`) is a
**field-keyed draft state machine**. The core is a `FIELDS` table
(`EnrollController.ts:136`) — one `FieldHandler` per settable field:

```
const FIELDS: Record<CharGenField, FieldHandler> = {
  species: { applicable, isSet, options, validate, apply },
  sex:     { … },
  name:    { … },
  pronouns:{ … },
  aspiration: { … },
};
```

`CharGenField` is `'species' | 'sex' | 'name' | 'pronouns' | 'aspiration'`
(`types/src/index.ts:936`). A `FieldHandler` knows how to: decide if the
field `applicable` to the current draft (sex only applies to a sexed
species), report whether it `isSet`, produce its option list, `validate`
a value, and `apply` it to the draft.

`execute` (`EnrollController.ts:479`) dispatches on the rest-of-line:

- **bare `enroll`** → re-emit the full draft state (no mutation).
- **`enroll <field> <value>`** → `FIELDS[field].validate` then `apply`,
  then re-emit the full state. This is **live-fire**: each field-set is
  its own command, applied and reflected immediately.
- **`enroll confirm`** → gate on `computeMissing(draft, cfg)`
  (`EnrollController.ts:272` — the `FIELD_ORDER` fields that are
  applicable but not yet set). If anything is missing, emit a
  field-scoped error; otherwise `commit`.

There is **no `currentStep`, `activeStep`, `goBack`, or `promptFor`** —
those belonged to the retired step model. Navigation is the client's
job. `species.apply` is idempotent: re-setting the same species is a
no-op, but *changing* species clears the now-stale downstream picks
(name/surname/sex) and refreshes the suggestion.

### Idempotent / order-free semantics

Because field-sets are commands and commands from one giver can arrive
back-to-back, the substrate is order-free and idempotent. The pipeline
also depends on **per-socket inbound serialization** (`Backend.ts`): a
single client's messages process in arrival order, never interleaved.
Without it, two concurrent `enroll` commands would each clone the one
`EnrollController` template and the second would trip
`StuffApi.clone`'s in-flight cycle guard. See
[connection.md](./connection.md).

## The wire payload

Every `execute` ends by emitting a `session.identity.state` frame
carrying a `CharGenStatePayload` (`types/src/index.ts:1004`) — the whole
picture, every time:

```
interface CharGenStatePayload {
  picks: CharGenPicks;            // what's chosen so far
  speciesOptions: CharGenOption[];
  sexOptions: CharGenOption[];    // empty ⇒ field N/A (non-sexed species)
  pronounOptions: CharGenOption[];
  aspirationOptions: CharGenOption[];
  suggestion?: { name; surname? };
  accountName?: string;
  missing: CharGenField[];        // drives the client's gating
  error?: { field; message };
}
```

The payload is **layout-agnostic**: option lists for every field, the
accumulated picks, and the `missing` set. The client renders whatever
screens it wants from this — it never asks the server "what step am I
on?" A `CharGenOption` carries `value`/`label`/`description`/`image` and,
for species, a `SpeciesDossier` (below).

## Commit + spawn

`commit` (`EnrollController.ts:620`) is the only step that persists. The
sequence, in order:

1. **Build the per-character overlay** (the picks over the shared seed
   at `Avatar.SEED_TEMPLATE_PATH`). **No per-player template row is
   written** (the identity doctrine — ref-shapes.md § Identity,
   lineage, and backing): the character's durable state is its
   persistence-spine snapshot, not `domain` data.
2. **Register ownership — the atomicity boundary.** Push `playerId`
   onto `user.playerIds` and `user.save()`. Nothing before this entered
   the roster, so a crash mid-char-gen leaves no trace.
3. **Clone the runtime Avatar from the SHARED seed** with the overlay
   riding `StuffApi.clone`'s `dataOverlay` and the minted identity path
   via `asTemplatePath` (`postRegister` stamps identity, installs the
   baseline implant, self-places at the app-config spawn, and captures
   the FIRST snapshot — the moment the character becomes durable).
4. **Set sex** on the live avatar (species-constrained, so applied after
   the species is in place to avoid hydration-order coupling).
5. **Dress in the aspiration's themed outfit** — best-effort: each
   garment is cloned, moved onto the avatar (`ContainmentApi.move`), and
   its body-plan slot claims occupied (`SlotApi.occupyAll`), narrowed
   with `MixinApi.isContainable`/`isWearable`/`isSlotted`. Missing
   garments or slot mismatches are skipped (content may lag).
6. **Hand off**: `ConnectionApi.transfer(interactive, avatar)`, then
   `avatar.enter(interactive, { firstArrival: true })`, then destruct
   `Login`.

`Avatar.enter`'s `firstArrival` flag selects the greeting: a fresh
"Welcome, *name*." for a just-created character vs. "Welcome back,
*name*!" for a returning login (`Avatar.ts`). On the client, entering
the world from char-gen (or the roster) clears the terminal buffer so
the `enroll …` echoes don't bleed into the world; a reconnect keeps its
scrollback (`store/index.ts` `setConnected`).

## Species substrate consumed by char-gen

`Species` (`lib/species/Species.ts`) is a `Visible` template (it speaks
the standard `shortDescription`/`longDescription` surface — the old
`defaultDescription` was subsumed — and carries an `illustration` key,
the kind's portrait). char-gen reads:

- `commonNames` — the roster keys/labels (`'human'`, `'elf'`, …).
- `sexDeterminationSystem` — non-empty ⇒ the species is sexed ⇒ the
  `sex` field applies and offers male/female.
- `nameBankKeys` — references to one or more `NameBank` Documents that
  feed the name suggester.
- `illustration` — the bucket-relative key for the species portrait,
  surfaced on `CharGenOption.image` (the 3:4 detail-pane slot). The client
  resolves it via `MEDIA_BASE_URL`; see [media.md](./media.md).
- `getBodyPlanPath()` / Material path — drive the **species dossier**:
  the scientific binomial, the Linnaean classification ladder (derived
  from the taxonomic template path), and anatomy/composition lines. This
  is the showcase of the depth of the world model, rendered in the
  species detail pane.

**The dossier and the illustration are presentation, not controller
logic.** The dossier is built by `SpeciesApi.buildDossier(species, path)`
(`api/species.ts`) — a readout of the species model, not picker code;
`EnrollController` only pre-warms a per-species card (`{ dossier,
illustration }`) and surfaces it, staying about draft state (read the
choices, write the picks).

### The name suggester + `NameBank`

`NameBank` (`lib/species/NameBank.ts`) is a `Document` in its own
`name_banks` collection — **not** inlined on the `Species` template (a
shared "common human" bank can back several species). It holds `given`,
`surname`, and `style` pools. `NameBank.resolve(keys)` unions the pools
for the species' `nameBankKeys`. Banks are installed from the
`@saxonberg/content-species-and-names` content pack
(`content/name-banks/<key>.yaml`, file name = bank key) by the `PackApi`
`name-banks` content kind — see
[content-packs.md](./content-packs.md).

The suggester runs on `Species`: `suggestName(realName)` biases the
pick by the player's real/account name (e.g. same initial), and
`rerollName()` produces a fresh draw. `EnrollController.refreshSuggestion`
regenerates the suggestion onto the draft on species-change and on
`enroll name reroll`. Names support keep / re-roll / type-your-own and
reject values failing the validation rules (length, Unicode letters,
single internal hyphen/apostrophe, no digits/spaces).

### `PersonaMixin`

`PersonaMixin` (`lib/character/Persona.ts`) carries the **claimed
self-narrative** layer — two persistent fields on `Character` (so PCs
and any future storied NPC have them): `bio` (free-form authored prose)
and `aspiration` (the closed-choice of who you're becoming). The
aspiration seeds the starter `bio`, selects the themed outfit, **and
seeds the chronicle's `claim` prologue** (a parallel `claimSeeds` field,
distinct from `bioSeed`) at commit. `Persona` also affords the
**`chronicle`** self-view verb (the partitioned bio → prologue → deeds
readout — see [chronicle.md](./chronicle.md)). `Persona` is **not** for
witnessed deeds (those are chronicle `deed`s), perceived body description
(`Visible`), or proper-name identity (`Named`). `bio` *editing* still
defers to Wave 2 (a `records` verb); the deferred `records` *view* role
shipped as `chronicle`.

## The cockpit phases (client)

The client `connectionPhase` (`store/index.ts:47`) is a four-state
machine: `'unauthenticated' | 'character-select' | 'char-gen' |
'in-world'`. The `session.identity.roster` frame routes to the
character-select roster; `session.identity.state` flips to the
char-gen stage and drives it; `session.link` (always
carrying an avatar) is the unconditional in-world flip.

`CharGenStage.tsx` owns the layout. A `SCREENS` config
(`CharGenStage.tsx:457`) chunks the fields into screens —
species · sex+pronouns · name · aspiration · review — with client-side
Back/Next navigation. Card clicks live-fire `enroll <field> <value>`;
the name field flushes `enroll name <given> <surname>` on blur/Next
(deduped); the review screen's confirm fires `enroll confirm`. Continue
and Back are pure client pagination and send no command. Because the
server is layout-agnostic, the whole screen set is reconfigurable
client-side (the single-page A/B variant).

## History

The intake substrate shipped in the char-gen Wave 1 build
(branch `feature/char-gen-wave1`, GitLab #21 / MR !49). The original
build used a server-driven **step model** (`ENROLL_STEPS` / `currentStep`
/ `activeStep` / `goBack`); a late re-architecture (commit `6942674`)
replaced it with the field-keyed draft state machine + client-owned
layout documented here, so single-page vs. multi-step is a pure client
concern. That commit also added per-socket inbound serialization (the
concurrency fix the single-controller dispatch surface needs) and the
first-arrival greeting / terminal-clear-on-handoff polish.

Deferred to later waves: the `records` verb (bio *editing* — its *view*
role shipped as the `chronicle` verb, see [chronicle.md](./chronicle.md)),
vitals/language, the lounge and onboarding flow, and the name sanitizer
(the denylist is a seeded stub today). (Breadcrumbs were renamed and
shipped as the chronicle identity ledger.)
