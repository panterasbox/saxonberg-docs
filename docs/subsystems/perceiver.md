# PerceiverMixin

Owns the verbs of perception: `look`, `scry`, `locate`, `find`, the
five sense verbs (`smell`, `listen`, `feel`, `taste`, `sense`), and
`assess` (a body's condition/wounds — medicine-gated detail out of
combat, the costed fog-graded tactical read mid-fight). Composed on
`Character`, so every Avatar and NPC inherits the perception verb
surface.

The split is by responsibility. Three mixins co-compose on
Character:

- **`Sensor`** (`lib/message/Sensor.ts`) — receives scene output.
  `handleMessage(frame)`. The substrate any host needs to be on
  the receiving end of `MessageApi.scene(...)`.
- **`Visible`** (`lib/description/Visible.ts`) — can be perceived.
  Owns descriptions and keywords. Contributes **no verbs** — pure
  target shape.
- **`Perceiver`** (`lib/description/Perceiver.ts`) — issues
  perception verbs. Contributes `look` / `scry` / `locate` / `find`
  plus the five sense verbs (`smell` / `listen` / `feel` / `taste` /
  `sense`) and `assess` on the **actor-side** bucket (`self`).

The split fixes a semantic conflation: `Visible` used to contribute
`look` to both `self` and the target-side buckets
(`environment` / `inventory` / `peers`). The `self` part read as
"I can be looked at therefore I can look" — the role inversion
that motivated splitting Perceiver out. The target-side part read
as "a visible thing nearby grants me the verb to look" — a
subtler inversion: the actor's capability shouldn't come from a
target's existence. Compare a `Throne` contributing `sit` on
`environment`: that one is correct because `sit` only exists as a
verb-against-that-specific-target, whereas `look` is a
perceiver-side verb that takes any reachable Visible as a target
via scope resolution. So Visible drops verb contributions
entirely; Perceiver is the sole source of `look`.

## Composition

`Perceiver extends Sensor` (interface-level). At runtime, every
Perceiver also composes Sensor — the perception verbs render
descriptions and send output back through the perceiver's own
Sensor channel. Documenting the prereq at the type level lets
consumers narrowing via `MixinApi.isPerceiver(host)` reach the
Sensor surface without a second narrow.

Composition slot on Character (innermost to outermost):

```
Named → Organism → Slotted → BodyPlanSlots → Posed → Gendered →
Sensor → Perceiver → Perception → Vocal → Soul → Visible →
Containable → Container → Engaged → Mobile → CommandGiver
```

`Perceiver` sits adjacent to `Sensor` so the chain reads
"perceiver layer right above the sensor substrate."

## Verbs

| Verb | Bucket | Lives where | Gated by |
|---|---|---|---|
| `look` | `self` | PerceiverMixin | — |
| `scry` | `self` | PerceiverMixin | — |
| `locate` | `self` | PerceiverMixin | — |
| `find` | `self` | PerceiverMixin | — |
| `smell` | `self` | PerceiverMixin | `requiresSmell` |
| `listen` | `self` | PerceiverMixin | `requiresHearing` |
| `feel` | `self` | PerceiverMixin | `requiresTouch` |
| `taste` | `self` | PerceiverMixin | `requiresTaste` |
| `sense` | `self` | PerceiverMixin | — (gestalt) |

All nine are actor-side only. The five sense verbs are each gated by
a `requires*` sensorium validator (see
[senses.md](./senses.md)); `sense` is the gestalt form, filtered to
the viewer's full sensorium rather than one channel. The verb lands
on the giver's stack
because they're a Perceiver, not because there happens to be a
visible / scryable / locatable thing in scope. Target resolution
runs at execution time through the verb's scope rules, narrowing
to whatever Visible / Scryable / Locatable thing the binding
resolves to.

### `read` — the marks substrate (`MarkedMixin`)

`read` (`cmd/perception/read.yaml`) is afforded by **`MarkedMixin`**
(`lib/description/Marked.ts`, registered `Marked`). A mark carries two
**independent axes**: `form` — how it is made, inked vs embossed — and
`markScript` — what system it is in, over a `MARK_SCRIPTS` vocabulary with
a `COMMON_SCRIPT` default. **`read` decomposes into perceive + decode**:
inked text needs light; embossed text reads in the dark by touch, so the
modality falls out of the form. v1 has exactly one script and no literacy;
a literacy gate slots into `decode` without disturbing `perceive` (the
spellbook comprehension floor is already a decoding gate in all but
name). ⭐ Collapsing form and script would make braille and a ciphered
dispatch inexpressible — the symmetry (embossed common lettering reads by
touch *and* vision) is why the split ships before any literacy.
(Graduated from the language slate, 2026-09.)

## Methods

⭐⭐⭐ **The surroundings read — and why it is not a hook.**

| member | kind | who calls it |
|---|---|---|
| `learnSurroundings(location, occupants?)` | method | a verb that is describing a place |
| `learnTimetable(stops)` | method | a verb that just rendered a public board |

`learnSurroundings` runs the perception gate
(`obviousExitsFor(viewer)`), records who the viewer saw
(`learnIdentityOf`, when the host is a `BeliefStore`), records where
they are and the ways out (`recordSurroundings`, when the host is a
`Cartographer`), and **returns the gated exits** — the same list the
verb renders, so the two cannot disagree.

⚠⚠ **It reaches each recorder DIRECTLY, narrowed by `MixinApi.isX`**,
and that is the project's settled shape for *record what you
perceived*: `BeliefStore.learnIdentityOf` and
`PerceptionApi.recordDiscovery` both predate this and both work that
way.

⛔ **There were two optional `@hook`s here and they are gone.**
`onPerceivedPlace` and `onReadTimetable` had **one implementer between
them** (`CartographerMixin`), and the generality claimed for them was a
hypothetical second map-keeper — the same use case. An optional hook
also cannot be invoked without a structural cast, which is where three
controllers' casts came from: `look`, `look`-in-the-dark and `sense`
each fired the hook through its own, each re-assembled the gate and the
`Exitable` narrowing, and they had **already diverged** (`look`
recorded who it saw, `sense` did not — so the one verb an arriving body
is forced into noticed the room and nobody in it).

⭐⭐ **A hook earns its keep by having more than one implementer.**
`Mobile.onTraversed` does — the cartographer and `RespirationMixin`
both implement it — so it stays a hook, fired on the mover from inside
the move, and no controller has ever needed to know the cartographer
exists. These two did not, so they became calls.

⛔ **And the naming was wrong.** This was `perceivePlace`. What gets
recorded is **navigational** — *I was here*, *this way leads there* —
not perceptual, so the method is `learn*` to sit beside
`learnIdentityOf`, and *"surroundings"* is the game's own word for the
room-level read (`"Your surroundings are indistinct."`). The gate
itself genuinely is perception, which is why this lives on `Perceiver`:
a hidden exit is **absent** from the list rather than filtered out
later, and that is what makes the map's evidence firewall structural.
See [location-graph.md](./location-graph.md) for the claim channels
(`walked` · `seen` · `published`) and the two sense-shaped fields that
were deleted with the old vocabulary.

## Scryable

`lib/perception/Scryable.ts`. The capability seam for `scry`:
in-fiction instruments (mirror, crystal ball, telescope) compose
`ScryableMixin` and override `canScryFor(target): VetoResult` to
gate which targets the instrument permits.

`Scryable extends Visible` because scry renders the target's
description surface (`getShort` / `getLong` / etc.) — anything
scryable is by definition visible.

v1 ships no concrete instruments. The mixin and interface exist so
content authors can compose them onto a `Mirror` or `CrystalBall`
class without a one-off interface dance; `ScryController` finds
candidates via `MixinApi.isScryable(item)` over the avatar's
environment + inventory.

## See also

- [shell-author.md](./shell-author.md) — `teleport` is admin /
  world-manipulation; `goto` is locomotion-of-self. Neither is a
  perception verb.
- [shell-workspace.md](./shell-workspace.md) — sibling shell-tier
  mixin; doesn't intersect with Perceiver.
- [command-routing.md § Affordance attribution](./command-routing.md) —
  PerceiverMixin's verb contributions are an actor's innate
  affordances; the Scryable seam above is the same mechanism with a
  wielded instrument as the source. A verb can be afforded by many
  source objects (innate `'self'`, instrument, future skill / implant);
  the source object — not a category enum — is the discriminator.
