# Quantities

The `Quantity<U>` value object is Saxonberg's substrate for engine
state that carries real units. Light intensity (lux), luminous flux
(lumen), color temperature (Kelvin), mass (kg), molar mass (g/mol),
and density (kg/m³) all flow through `Quantity` rather than through
opaque scalars. Future channels (sound, heat, electricity) plug
into the same shape.

The substrate ships:

- A `Quantity<U extends Unit>` value class with construction
  factories, same-unit math, comparison, cross-unit conversion
  (where defined), tag-table integration, JSON round-trip, and Mml
  emission via `<quantity>` markup.
- A `Unit` string-literal union — the v1 catalog of allowed units.
- A tag-table registry (`Quantity.registerTagTable`) consulted by
  `tag()` / `fromTag()` / `parse()`.
- A per-unit math op table (`add` / `scale`) so future non-arithmetic
  units (`dB` for sound) plug in without retrofitting callers.
- `QuantityMarshaller<U>` for round-tripping Quantity values through
  the persistence boundary.

Sibling docs:

- [light.md](./light.md) — first major consumer. Light's `intensity`
  is `Quantity<'lux'>`, `colorTemperature` is `Quantity<'K'>`,
  emitter / ambient APIs return Quantity values.
- [race.md](./race.md) — Material's `density` (`Quantity<'kg/m³'>`),
  `molarMass` (`Quantity<'g/mol'>`), and TangibleMixin's `mass`
  (`Quantity<'kg'>`).
- [persistence.md](./persistence.md) — Marshaller framework, the
  `QuantityMarshaller` story, and PropertiedMixin's
  `savedPropMarshallers` for per-prop binding.

## The cast

| Type | Kind | Role |
|---|---|---|
| `Quantity<U>` | value class | `value: number` + `unit: U`. Immutable. Same-unit math; cross-unit conversion via registered converters. |
| `Unit` | string-literal union | The full v1 unit catalog. New units extend the union. The Vitals build added `bpm` (rate; pulse + respiration via separate tag scales, no converter — its own axis), `mmHg` (pressure, `mmHg ↔ Pa` converter), and `L` (volume, `L ↔ m³` converter), plus `W/(m·K)` (thermal conductivity — a real `Material` property; `encodeUnit` strips `()`/`·` so it marshals to `W-per-m-K`). The Bulk build added `mL` and `cup` (liquid volume; `mL ↔ L` and `cup ↔ L` converters; **tagless** like `m³` — the bulk substrate stores amounts canonically as `L`). The Metabolism build added `mg` (sub-gram mass; `mg ↔ g` converter — toxin/nutrient authored amounts + absorbed-dose math) and `g/dL` (blood-alcohol concentration; carries the **`'bac'` drunk-ladder scale** — `sober`/`tipsy`/`drunk`/`very-drunk`/`incapacitated`/`life-threatening` at ascending thresholds — which `getBAC()` reads live for the `intoxication` condition's severity; registered in-file via `Quantity.registerTagTable` after the class declaration, not from a channel module). The Thermal build added `J/(kg·K)` (specific heat — a real `Material` property; marshals to `J-per-kg-K`), `clo` (clothing insulation, its own additive axis — on `WearableMixin`), and `m/s` (wind speed — a static-authored `Biome`/`Atmospheric` field), all **tagless**, and exercises the `K` **`'thermal'` scale** (frozen/cold/cool/warm/hot/scorching) shipped with surface-architecture. The Electricity build added `V` / `A` / `Ω` / `S/m` (the Ohm's-law triple + `S/m` = electrical conductivity, a real `Material` property; `S/m` marshals to `S-per-m`) plus the coursework sub/super scales `kV`/`mV`/`mA`/`kΩ`/`MΩ` (converters to the SI base), with banding tables for `A` (the physiological current ladder), `Ω`, and `S/m`. Adding a unit = extend the union + `unitOps` + (optional) a `quantity-tags.yaml` block + extend the schema's `patternProperties` regex (only if it has tags) + a per-unit `QuantityMarshaller` seed. |
| `ScaleName` | string alias | Names a tag-vocabulary scale within a unit (e.g. `'default'`, `'color'`, `'thermal'`). Pure rendering choice — units are units regardless of scale. |
| `TagTableEntry` | `{ tag, threshold }` | A single row in a unit's tag table. |
| `QuantityUnitMismatchError` | error | Thrown by same-unit math when a runtime cast bypassed the compile-time guard. |
| `QuantityMarshaller<U>` | `Marshaller` subclass | Round-trips a Quantity field through the persistence boundary. One singleton per unit, registered at `pathFor(unit)`. |

## The Quantity API

```ts
class Quantity<U extends Unit> {
  // Inspection
  rawValue(): number
  format(): string                                   // "5 kg", "320 lux"
  tag(scaleName?: ScaleName): string                 // "medium" / "lit" — falls back to format() for tagless units
  toJSON(): { value, unit }
  toMml(viewer?, scaleName?: ScaleName): Mml         // <quantity>tag-text</quantity>
  formatMml(viewer?, scaleName?: ScaleName): Mml     // <quantity>5 kg</quantity>

  // Math (same-unit only; throws on mismatch)
  add(other): Quantity<U>
  subtract(other): Quantity<U>
  scale(factor: number): Quantity<U>

  // Comparison
  equals(other): boolean
  lessThan(other): boolean
  greaterThan(other): boolean

  // Cross-unit conversion — returns Quantity<Unit> (the target is runtime-parametric)
  to(targetUnit: Unit): Quantity<Unit>

  // Construction (private constructor; entry shapes below)
  static of<U>(value: number, unit: U): Quantity<U>
  static parse<U>(input: string | number, unit: U, scaleName?: ScaleName): Quantity<U>
  static fromTag<U>(tag: string, unit: U, scaleName?: ScaleName): Quantity<U>
  static fromJSON<U>(json: { value, unit }): Quantity<U>

  // Tag-table registration (`QuantityApi.loadTagTables` calls this
  // for every (unit, scale) pair declared in the YAML at boot)
  static registerTagTable(
    unit: Unit,
    scaleName: ScaleName,
    entries: ReadonlyArray<TagTableEntry>,
  ): void
  static setDefaultScale(unit: Unit, scaleName: ScaleName): void

  // Tag-table introspection — generic banded-arithmetic substrate
  // for consumers that need to do shift / compare math over a
  // unit's registered ordering (e.g. light-band perception shifts).
  static tagsFor(unit: Unit, scaleName?: ScaleName): readonly string[]
  static shiftTag(unit: Unit, tag: string, shift: number, scaleName?: ScaleName): string
  static compareTag(unit: Unit, a: string, b: string, scaleName?: ScaleName): number
}
```

### Construction shapes

`Quantity.of(value, unit)` is the canonical factory. The tolerant
shapes go through `parse`:

| `Quantity.parse(input, 'kg')` | Result |
|---|---|
| `'heavy'` | `Quantity.of(50, 'kg')` (KG_TAGS lookup) |
| `'5 kg'` | `Quantity.of(5, 'kg')` |
| `'12000 g'` | `Quantity.of(12, 'kg')` (g↔kg converter) |
| `'5'` or `5` | `Quantity.of(5, 'kg')` (bare-number → canonical-of-target) |

### Same-unit math

`add` / `subtract` / `scale` operate on the unit's registered op
table. v1 ships only arithmetic (numeric add and scalar multiply),
so mass + mass and lux + lux work as expected. `dB`'s logarithmic
addition is the one entry deferred until sound lands; a future
registration on `unitOps` will replace `add` for `'dB'` without
touching call sites.

```ts
Quantity.of(5, 'kg').add(Quantity.of(3, 'kg'));        // → 8 kg
Quantity.of(5, 'kg').add(Quantity.of(3, 'g') as any);  // → throws QuantityUnitMismatchError
Quantity.of(5, 'kg').scale(2);                         // → 10 kg
```

The mismatch error surfaces when a runtime cast bypasses TS's
compile-time guard. Code that types `Quantity<'kg'>` correctly
won't see it.

### Cross-unit conversion

`to(targetUnit)` consults the converter registry. Each pair is
declared explicitly in `lib/quantity.ts`. v1 registers `g ↔ kg` for
mass authoring; cross-family conversions throw.

The return type of `to` is `Quantity<Unit>` (not narrowed to the
target). Callers that need narrow typing should rely on host-level
typed accessors (e.g. `light.intensity` is statically
`Quantity<'lux'>`) rather than chaining through `to`.

## Tag tables

Tag tables are pure data — the API forces every consumer through
`Quantity.tag()` / `Quantity.fromTag()` / `Quantity.parse(tag)`, so
the tables don't need to live in TypeScript. They live in a single
content-authorable YAML at `mud/config/quantity-tags.yaml`,
validated against `mud/config/quantity-tags.schema.json`.

The registry is double-keyed by `(unit, scaleName)`. Most units
ship a single scale named `default`; units that carry multiple
authoring vocabularies for the same numerics (notably `K`, where
color-temperature and thermal-temperature read with different
tags) register one scale per vocabulary.

```yaml
# mud/config/quantity-tags.yaml (excerpt)
kg:
  default:
    - { tag: feather,  threshold: 0.001 }
    - { tag: light,    threshold: 0.5 }
    - { tag: medium,   threshold: 5 }
    - { tag: heavy,    threshold: 50 }
    - { tag: enormous, threshold: 500 }

K:
  color:
    - { tag: warm,     threshold: 2700 }
    - { tag: neutral,  threshold: 4000 }
    - { tag: cool,     threshold: 5000 }
    - { tag: daylight, threshold: 6500 }
    - { tag: blue,     threshold: 10000 }
  # A future thermal scale slots in here:
  # thermal:
  #   - { tag: frozen,  threshold: 0 }
  #   - { tag: room,    threshold: 295 }
  #   - { tag: boiling, threshold: 373 }
```

`QuantityApi.loadTagTables()` reads + validates the YAML and calls
`Quantity.registerTagTable(unit, scaleName, entries)` for every
declared `(unit, scale)` pair at boot — `AppBootstrap` runs it
after `SeederManager.run()` so the tables are available before
any code that hits `tag()` / `parse(tagString)` runs (the
marshallers, the propagation walks, controllers).

### Scales

Scales are RENDERING choices, not type distinctions. `Quantity<'K'>`
carries no semantic — the same instance can render with EITHER
scale at the call site:

```ts
const q = Quantity.of(3000, 'K');
q.tag();          // → 'warm'    (default scale: 'color')
q.tag('color');   // → 'warm'
q.tag('thermal'); // → 'boiling' (if thermal scale registered)
```

The pedagogical principle: science students playing the game know
"a unit is a unit is a unit." Color temperature for a blackbody
emitter IS thermal temperature — the engine refuses to fork the
type system on what is, fundamentally, vocabulary preference. Math,
equality, and conversion are unit-only.

**Default scale.** The first scale registered for a unit becomes
its default; `Quantity.tag()` / `parse()` / `fromTag()` consult it
when no scale name is passed. Author the intended default first
in the YAML, or call `Quantity.setDefaultScale(unit, scaleName)`
to switch deliberately. Removing the default scale via reload
auto-promotes the next remaining scale.

The lookup machinery in `lib/quantity.ts`:

- `Quantity.tag(scaleName?)` walks the resolved table descending;
  first threshold met-or-exceeded wins. Falls back to canonical
  format when no table is registered for the (unit, scale) pair.
- `Quantity.fromTag(tag, unit, scaleName?)` linear-scans for the
  tag and returns its threshold as the canonical numeric value.
- `Quantity.parse(input, unit, scaleName?)` accepts numerics,
  literals, and tag strings; the optional scale name only affects
  the tag-string path.
- Round-trip stability — every `{ tag, threshold }` row satisfies
  `fromTag(tag, unit, scale).tag(scale) === tag`. Tests pin this.

### Banded-arithmetic helpers

`tagsFor` / `shiftTag` / `compareTag` expose the registered
ordering as a substrate for callers that need to shift or compare
**by band**, not by raw numeric. The motivating consumer is
per-viewer light perception — a low-light-vision species shifts
the perceived `LightBand` up by N bands rather than scaling lux,
which is the right semantic ("nightvision sees one band better"
not "nightvision multiplies illuminance by 4×"). The same machinery
applies to any unit/scale pair with an ordered vocabulary (a future
thermal-K scale, a sound-dB band scale, etc.) — adding a per-channel
shift/compare helper to each Api is no longer required.

The compile-time vocabulary type union (e.g. `LightBand`) and the
runtime registry are kept in sync by an `as const` tuple at the
type-source side and the `bandFor`-style adapter on the api side
(membership-checks the registered tag against the union; throws on
drift). See `lib/perception/Light.ts` for the canonical pattern.

### Reload

`QuantityApi.reloadTagTables()` re-reads the YAML and applies the
diff at `(unit, scaleName)` granularity: pairs present in the new
file get re-registered (replacing prior entries), pairs absent from
the new file get cleared. Existing `Quantity` instances see new
tags on their next `tag()` call — no caching layer to invalidate,
no Quantity-instance migration. Edits to band thresholds change
which band an already-stored value reads as; storage is unaffected
(Quantity values store numerics, not tag strings).

Removing a unit's default scale promotes the next remaining scale
to default automatically. Removing the last scale for a unit drops
the unit's registry entry entirely — `tag()` then falls back to
canonical format.

Production trigger for reload is a future admin slash command;
v1 calls it from a test harness when needed.

### v1 declared units

| Unit | Scale(s) | Tags |
|---|---|---|
| `lux` | `default` | `pitch-black`/`very-dim`/`dim`/`lit`/`bright`/`blinding` (mirrors `LightBand`) |
| `lumen` | `default` | `unlit`/`glow`/`lamp`/`bright`/`searchlight` |
| `K` | `color` | `warm`/`neutral`/`cool`/`daylight`/`blue` |
| `kg` | `default` | `feather`/`light`/`medium`/`heavy`/`enormous` |
| `kg/m³` | `default` | `gas-like`/`water-like`/`rock-like`/`metal-like` |
| `ppm` | `default` | `imperceptible`/`trace`/`faint`/`noticeable`/`pungent`/`overpowering` (smell concentration) |
| `dB` | `default` | `silent`/`faint`/`quiet`/`conversation`/`loud`/`piercing`/`deafening` (sound pressure level) |
| `Hz` | `default` | `sub-bass`/`bass`/`low`/`midrange`/`high`/`ultra` (frequency) |

Tagless units in the catalog (`g/mol`, `mol`, `mol/L`, `g`, `m`,
`s`, `Pa`, `N`, `J`, `W`, …) deliberately have no tag table — they
don't have a casual-prose vocabulary. `Quantity.tag()` falls back to
canonical format for them (`"55.845 g/mol"`).

### dB arithmetic — logarithmic add

The `dB` unit's `UnitOps` registry overrides `add` and `scale` to
implement decibel arithmetic correctly:

- `add(a, b) = 10 * log10(10^(a/10) + 10^(b/10))` — incoherent-sum
  formula. Two 60 dB sources sum to ~63 dB, not 120.
- `scale(value, factor)` — multiplying linear amplitude by a factor
  corresponds to adding `10 * log10(factor)` to the dB value;
  `factor === 0` resolves to a practical -120 dB floor.

The sound propagation walk in `SoundModality.signalAt` uses LINEAR
amplitude internally (sums `10^(dB/10)` terms across contributing
sources, applies the conduit transmissivity as a multiplier on the
linear scale) and converts back to dB at the wrap step — the
physically-correct merge for incoherent sources.

## Mml emission — `<quantity>` markup

`toMml()` and `formatMml()` produce `<quantity>` markup the client
will render with channel-coded styling, tooltips, alternate-flavor
toggles, etc. v1 ships only the server side; the client renderer is
the v1 punch-list "Markup language semantic tags + client renderer"
item.

**Markup shape:**

```
<quantity unit="<unit>" value="<numeric>" [tag="<tag>"]>inner-text</quantity>
```

| Attribute | Required | Description |
|---|---|---|
| `unit` | yes | Canonical unit string from the `Unit` type. The client routes per-channel rendering off this. |
| `value` | yes | Canonical numeric as a decimal string. The client uses this for any conversion / comparison work — never parse the inner text. |
| `tag` | optional | Friendly tag if a tag table is registered. Absent for tagless units. |

**Inner text:**

- `toMml()` emits the **tag** (or canonical for tagless units). The
  default for prose — `${quantity}` in `Mml.compose` calls this.
- `formatMml()` emits the **canonical format** (`"5 kg"`,
  `"320 lux"`). Used by instrument readouts and analyze output where
  the canonical number is the point.

**Examples:**

```ts
Mml.compose`mass: ${Quantity.of(5, 'kg')}`
// → mass: <quantity unit="kg" value="5" tag="medium">medium</quantity>

Mml.compose`mass: ${Quantity.of(5, 'kg').formatMml()}`
// → mass: <quantity unit="kg" value="5" tag="medium">5 kg</quantity>

Mml.compose`molar mass: ${Quantity.of(55.845, 'g/mol').formatMml()}`
// → molar mass: <quantity unit="g/mol" value="55.845">55.845 g/mol</quantity>
```

Markup-unaware clients strip `<quantity>` and render the inner text
verbatim — no garbled output, no leaked attributes.

**Viewer threading:** `toMml(viewer?)` carries an optional viewer
parameter for forward compatibility with a deferred per-recipient
pedagogical-seam setting. v1 ignores it; the seam is wired in
`Mml.compose`'s lazy evaluation and `Scene.send`'s per-recipient
body materialization (see [messaging.md](./messaging.md)).

### Liquid prose filters

`ProseApi` registers two filters for content-authored prose:

```liquid
{{ q | quantity }}            <!-- tag-flavored, default for prose -->
{{ q | quantity_canonical }}  <!-- canonical-flavored, for readouts -->
```

Same `<quantity>` output as the Mml-side helpers; choice of filter
controls inner text only.

## Persistence

Quantity values round-trip through the persistence boundary as
`{ value, unit }` JSON. Two adoption shapes:

### First-class fields — a `marshaller` entry in `fieldMeta`

A class with a `Quantity<U>`-typed field declares the marshaller
binding next to its persistent fields:

```ts
class Material extends ... {
  private _density: Quantity<'kg/m³'> = Quantity.of(0, 'kg/m³');

  static fieldMeta: FieldMeta = {
    density:   { persistent: true, marshaller: QuantityMarshaller.pathFor('kg/m³') },
    molarMass: { persistent: true, marshaller: QuantityMarshaller.pathFor('g/mol') },
  };

  // accessor pair; setter strict on Quantity
  protected get density(): Quantity<'kg/m³'> { return this._density; }
  protected set density(value: Quantity<'kg/m³'>) {
    if (!(value instanceof Quantity) || value.unit !== 'kg/m³') {
      throw new TypeError(...);
    }
    this._density = value;
  }

  public getDensity(): Quantity<'kg/m³'> { return this._density; }
  public setDensity(value: Quantity<'kg/m³'>): void { this.density = value; }
}
```

The marshaller absorbs authoring-shape coercion (numeric, tag
string, alt-unit literal, JSON `{value,unit}`) at the persistence
boundary; runtime setters stay strict on `Quantity<U>`. Authors who
hold a raw number wrap via `Quantity.of(n, unit)` at the call site.

### PropertiedMixin props — `initProp(prop, { marshaller })`

For props that hold rich value objects, declare the marshaller via
`initProp`:

```ts
const mass = Property.of<Quantity<'kg'>>("mass");
avatar.initProp(mass, {
  transient: false,
  marshaller: QuantityMarshaller.pathFor('kg'),
});
avatar.setProp(mass, Quantity.of(5, 'kg'));
avatar.getProp(mass);  // → Quantity.of(5, 'kg')
```

The binding persists in `savedPropMarshallers` alongside `savedProps`,
so reload-after-restart re-applies the same marshaller without
redeclaration. See [persistence.md § Marshalled props](./persistence.md).

### YAML authoring shapes

A field bound to a QuantityMarshaller accepts these shapes in seed
YAML, all coerced to the runtime Quantity at hydration:

```yaml
data:
  mass: heavy           # tag — KG_TAGS lookup → Quantity<kg>(50)
  mass: "5 kg"          # canonical literal
  mass: "12000 g"       # alt-unit literal — converts via g↔kg
  mass: 5               # bare numeric — canonical-of-target → 5 kg
  mass: { value: 12, unit: kg }   # explicit JSON (tooling-generated)
```

`Quantity.parse(input, targetUnit)` is the underlying handler;
authors typically use whichever shape is most readable.

### Marshaller seeds and bootstrap

Each unit has a one-line seed YAML in
`seeds/lib/persistence/QuantityMarshaller/<encoded>.yaml`. The
marshaller class itself is stateless modulo its target unit; the
seed declares the unit in `data:`.

Marshallers are **lazy-loaded** — `StuffApi.singleton(path)` clones
the marshaller from its seed on first need (no bootstrap manifest
entries required). Mirrors how `clone()` resolves `hydratorClass`.

In tests, marshallers don't have a Mongo to clone from; tests
register them in-memory via `installV1QuantityMarshallers()` from
`lib/persistence/__tests__/quantity-marshaller-test-helpers.ts` —
call it in `beforeEach` paired with `StuffApi.clearAll()` in
`afterEach` for any test that exercises marshaller-bound fields or
props.

## Path encoding for marshaller singletons

Composite units (`g/mol`, `kg/m³`, `mol/L`, `m/s²`) carry
characters that make poor filesystem segments. The encoding maps
`'/'` → `'-per-'`, unicode `³` → `'3'`, unicode `²` → `'2'`, and
`'%'` → `'pct'`, giving the templatePaths:

| Unit | Marshaller path |
|---|---|
| `'kg'` | `/obj/persistence/QuantityMarshaller/kg` |
| `'g/mol'` | `/obj/persistence/QuantityMarshaller/g-per-mol` |
| `'kg/m³'` | `/obj/persistence/QuantityMarshaller/kg-per-m3` |
| `'lumen'` | `/obj/persistence/QuantityMarshaller/lumen` |
| `'lux'` | `/obj/persistence/QuantityMarshaller/lux` |
| `'K'` | `/obj/persistence/QuantityMarshaller/K` |
| `'Pa'` | `/obj/persistence/QuantityMarshaller/Pa` |
| `'%'` | `/obj/persistence/QuantityMarshaller/pct` |
| `'m/s²'` | `/obj/persistence/QuantityMarshaller/m-per-s2` |
| `'m'` | `/obj/persistence/QuantityMarshaller/m` |
| `'m³'` | `/obj/persistence/QuantityMarshaller/m3` |

Use `QuantityMarshaller.pathFor(unit)` at every declaration site
rather than hardcoding the encoded form.

**Every `pathFor(unit)` needs a seed row** at
`seeds/lib/persistence/QuantityMarshaller/<slug>.yaml`, or cloning any
template whose class marshals that unit dies at hydrate with
`Template not found` — **in live play only**, because unit tests
install marshallers by hand (`installV1QuantityMarshallers`) and so
can't see the gap. This is exactly how the fire build's `MJ/kg`/`J/kg`
and electricity's `A`/`Ω` left oak, water, iron, and copper
un-clonable in a running server while the whole suite stayed green
(found by the magic build's browser drive). The roster is now pinned
by `lib/persistence/__tests__/quantity-marshaller-seeds.test.ts`,
which greps every `pathFor` literal in src against the seeds dir —
adding a unit without its seed fails CI.

### K — two registered scales (color + thermal)

`K` is the canonical multi-scale unit: the same `Quantity<'K'>` can
render as either a color-temperature tag (`warm` / `neutral` /
`cool` / `daylight` / `blue` — the lighting subsystem's vocabulary)
or a thermal-temperature tag (`freezing` / `cold` / `cool` / `warm`
/ `hot` / `scorching` — the biome subsystem's vocabulary).

`color` is K's default scale (registered first); biome instruments
opt into thermal rendering at the call site:

```ts
q.tag()              // color-scale tag (e.g. 'warm' at 2700 K)
q.tag('thermal')     // thermal-scale tag (e.g. 'warm' at 293 K)
q.formatMml(_, 'thermal')  // formatted with thermal tag attribute
```

The pedagogical principle: units are units. The engine doesn't
distinguish "color K" from "thermal K" at the type system; the
YAML supplies prose vocabulary for the relevant context.

The biome substrate adds tagless or thermal scales for
`Pa` (default), `%` (default), and `m/s²` (default) plus the
`thermal` scale on K. `m` and `m³` stay tagless (rendered
numerically). See [biome.md](./biome.md) for the breakpoints.

## Out of scope (v1)

- **Pedagogical-seam setting** — the per-viewer "show tag vs
  show canonical vs show raw number" toggle. `toMml(viewer?)`
  carries the parameter for forward compat; v1 ignores it.
- **Mixed-unit derivation** — `Quantity<m> / Quantity<s> →
  Quantity<m/s>`. v1 only does same-unit math; mixed-unit derivation
  arrives when consumers need it.
- **Composite units beyond what's used** — `mol/L`, `Pa`, `N`, `J`,
  `W` are declared in the `Unit` type for forward compat; only
  `kg/m³` and `g/mol` are actually consumed today.
- **Tag tables in YAML** — content-authorable tag tables are a
  future ask; v1 ships TS tables co-located with each channel's
  Api.
- **Locale-aware formatting** — `format()` always emits SI display
  (`"5 kg"`, never `"5 lb"`).
- **Instrument calibration / accuracy** — Photometers and Balances
  always report exact values. Per-instrument tolerance is a future
  axis.

## Cross-references

- [light.md](./light.md) — first major Quantity consumer.
- [race.md § Material substrate](./race.md#material-substrate) —
  `density`, `molarMass`, `mass`.
- [persistence.md § Marshaller framework](./persistence.md#marshaller-framework)
  — Marshaller base class, a `marshaller` entry in `fieldMeta`, lazy resolution.
- [messaging.md](./messaging.md) — `Mml.compose` lazy evaluation,
  `Scene.send` per-recipient body materialization (the substrate
  the viewer-threading rests on).
- [command-spec.md](./command-spec.md) — Option E (per-subcommand
  controllers), the `analyze` / `measure` verb pattern that
  consumes Quantity output.

## ⭐ A reading is more than a number

`toMml` / `formatMml` take an optional `QuantityMarkupOptions` carrying
four facts a bare value does not have:

| Field | Meaning |
|---|---|
| `channel` | *What kind of reading this is* — the client's stable identity for pinning to an instrument panel or charting a series. From `MEASURE_CHANNELS` (`lib/perception/MeasureChannel.ts`). |
| `via` | The instrument that produced it. |
| `lo` / `hi` | Optional working range, in this quantity's own unit. |

All four are **omitted from the emitted tag when absent**, so every
existing call site is unaffected and a quantity with no instrument
behind it never claims to have one.

`via` is the pedagogical seam: **a reading is only honest if you can say
how it was taken.** A number with no provenance is a number the player
has to trust; a number with an instrument beside it is one they can
reason about.

`buildMarkup` is the single chokepoint — the attributes thread through
the two public emitters rather than being written at every call site,
which is why adding them touched ~25 emission sites without editing any
of them individually. Attribute order is fixed (`channel`, `unit`,
`value`, `tag`, `via`, `lo`, `hi`) so snapshot assertions are stable,
and every value is escaped — `via` is the one that can carry authored
text, so escaping there is load-bearing rather than defensive.

⚠ `MEASURE_CHANNELS` is **not** `lib/material/Channel.ts`. That
vocabulary (`edge` · `point` · `blunt` · `shock` · `heat`) is the shape
of a *force*. These are kinds of *reading*. Homonyms at unrelated
layers — see [messaging.md](./messaging.md) for the full note and the
security posture of the `<quantity>` tag itself.
