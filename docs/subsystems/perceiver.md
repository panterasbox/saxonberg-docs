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

## Methods

v1 has none. The mixin's value is verb contributions and the
compositional marker. Methods may land later — e.g. a
`perceive(target)` for scripted NPCs to invoke programmatically
without going through the parser.

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
