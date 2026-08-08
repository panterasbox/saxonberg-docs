# App settings

Application-wide, operator-managed configuration: the values that are
*game-managed* (an operator changes them in-app and they persist) but are
*not* per-player, per-Stuff, secrets, or deploy-time infra. The fourth
config category, alongside per-player `settings`
([shell-environment.md](./shell-environment.md)), per-Stuff `Propertied`
([properties.md](./properties.md)), and server-persisted client-UI-state.

Source of truth for this subsystem.

## Shape

Three small pieces, modeled on the `WorldClockState` (Document) ↔
`WorldClockApi` (operations surface) split:

| Piece | File | Role |
|---|---|---|
| `AppSettings` | `lib/config/AppSettings.ts` | Singleton `Document` (`app_settings` collection) + the `AppSettingKeys` key vocabulary. Pure persistence + the warmed cache. |
| seed YAML | `mud/config/app-settings.yaml` | The single source of the settings' **values**. |
| `AppSettingsSeeder` | `backend/AppSettingsSeeder.ts` | Backend seeder (like `EmoteSeeder`/`ChannelSeeder`) — populates `app_settings` from the YAML. |
| `AppApi` | `api/app.ts` | The runtime read/write surface (and the future home for app-level ops). |
| `config` verb | `cmd/system/config.yaml` + `obj/command/system/ConfigController.ts` | Developer-gated in-app editing. |

### Storage is an open key/value bag, not a field per setting

`AppSettings` persists a *single* field — `values: Record<string,string>`,
so `persistentFields = ['values']` and never grows. Adding a setting is a
YAML entry plus a consumer; it never changes the Document's persisted
shape. The bag round-trips natively through Mongo (no marshaller). The verb
can set *any* key (open namespace).

### Values in YAML, keys in code — no code-side defaults

The settings' **values** live only in the DB, seeded from the YAML. There
is no code defaults map: a code constant duplicating the seeded values
would just be a second copy on disk of what the seeder already guarantees.

What stays in code is the **key vocabulary** — `AppSettingKeys` (in
`AppSettings.ts`) — so consumers reference a constant, not a bare string (a
typo is a compile error):

```ts
// lib/config/AppSettings.ts
export const AppSettingKeys = {
  defaultStartLocation: "defaultStartLocation",
  evacuationFallback: "evacuationFallback",
} as const;
```

```yaml
# mud/config/app-settings.yaml — the single source of the values
settings:
  - key: defaultStartLocation
    value: /domain/lounge/warren
  - key: evacuationFallback
    value: /domain/void
```

## The `AppApi` surface — runtime operations only

```ts
AppApi.setting(key): string              // sync cached read; values[key] ?? ""
AppApi.settings(): Record<string,string> // the whole bag, for the listing
AppApi.setSetting(key, value): Promise    // write + persist + refresh cache
```

- **Reads are synchronous** off the warmed cache — the evacuation path in
  `Container.cleanupOnDestruct` cannot `await`. The seeder guarantees the
  value is present, so reads just hit the cache — no code-side fallback; an
  unseeded/unknown key reads `""`.
- **Reads are ungated**: the internal consumers (evac, avatar-mint) are
  engine code, not developers. **`setSetting` is reached only through the
  developer-gated `config` verb**; the gate lives at the verb (its
  `requiresWizard` validator), not duplicated on the Api.
- **`setSetting` is self-refreshing**: the cache holds the very instance it
  mutates and saves, so there is no separate re-read.
- **No boot/seed/warm method.** `AppApi` is the home for app-level
  *operations* (settings now; `shutdown()`, MOTD, maintenance mode later —
  things an operator invokes). Seeding (`AppSettingsSeeder`) and the boot
  cache-warm (`AppSettings.warm`) are backend infrastructure (below), not
  Api methods — deliberately unlike `WorldClockApi.boot()`, because the
  clock's boot starts a running subsystem whereas app settings have nothing
  to start.

> **Not the backend `Application`.** `AppApi` (`mud/api/app.ts`) is the
> in-engine domain surface. The backend `Application` class
> (`packages/server/src/backend/Application.ts`) is server / OAuth / signup
> orchestration a layer down. Adjacent names, different layers.

## Seeding + warming is a backend bootstrap concern

**Seed.** `AppSettingsSeeder.run()` runs in `AppBootstrap`'s seeder block
(next to `EmoteSeeder`/`ChannelSeeder`), reading
`mud/config/app-settings.yaml`. It is insert / **merge-missing** /
idempotent: a fresh DB gets the seeded row; on later boots any key *new* to
the YAML is merged into the existing row, while keys an operator changed via
`config` are left alone. It lives in a per-collection seeder (not
`SeederManager`, which seeds *Stuff templates* into `domain`) — same
reasoning as `EmoteSeeder`.

**Warm.** The cache is held on the `AppSettings` Document (a `private
static` singleton slot). `AppBootstrap.run` `await`s `AppSettings.warm()`
once at startup — after the seeder has populated the row and before any
consumer reads a setting (the evac path can't `await`). `warm` only loads
the row into the cache (or an empty instance if nothing's been
seeded/set); it does **not** seed. Reading a setting before the warm step
throws loudly (`AppSettings.getCached`) rather than returning a silent
`undefined`.

## The `config` verb

Single-token `config`, in the `system` command category, gated by the
`/lib/command/validators/requiresWizard` validator (the `eval`/`reload`
precedent — operator/TS-escape tier):

```
config                                       list every setting + current value
config defaultStartLocation                  show one
config defaultStartLocation /domain/lounge   set one (persist + refresh cache)
```

Two optional positional args (`key`, `value`); not subcommands. `value` is
a single token (a setting value is a path/scalar) — not greedy, because a
greedy arg is treated as required and a required arg cannot follow the
optional `key`. The listing shows every setting and its current value,
flagging keys outside the `AppSettingKeys` vocabulary with `*`. Setting a
key not in that vocabulary succeeds and round-trips (open namespace) and
earns a soft prose note — prose only, status stays `ok` (the write
succeeded; it is **not** a `controller-rejected` envelope note).

## The two v1 settings and their consumers

| Key | Seeded value | Read by |
|---|---|---|
| `defaultStartLocation` | `/domain/lounge/warren` | The three avatar-mint sites stamp it into a new avatar's `startLocation` at clone time: `EnrollController.commit`, `Application.createDefaultAvatarTemplate`, `Login.mintRandomGuestAvatar`. |
| `evacuationFallback` | `/domain/void` | `Container.cleanupOnDestruct` — where an orphaned `HasInteractive` evacuates when its container destructs with no outer. |

`defaultStartLocation` supplies only the *initial* value of each avatar's
per-character `Avatar.startLocation` field (the durable spawn/recall home,
individually mutable thereafter). The Avatar seed YAML no longer carries a
`startLocation` literal — all three mint sites inject from app config, so
an operator can move the new-player spawn in-app without a deploy.

`evacuationFallback` defaults to the void deliberately: `VoidLocation` is
the bootstrap-pinned, destruct-refusing singleton built to be the evac
fallback — a more reliable live Container than any ordinary room. This
replaced the retired `config/constants.ts` constant, which conflated the
spawn default with the evac fallback and pointed evac at the (destructible)
lobby.

> Later builds added their own keys (reactions, forums anti-snowball,
> renown/participation/producer/influence/conviction, the `crafting.*`
> family — broken threshold + delivery floor, wear-per-use rates, the
> keenness dials, repair pricing, the salvage rate); each is documented in
> its own subsystem doc. The table above is the original v1 exemplar pair,
> not a live registry — `AppSettingKeys` is.

## The scattered-constant sweep

A pass over modules predating this subsystem lifted a handful of hardcoded
policy literals into the bag. The bar is deliberate: a candidate is a
**single-source** value an operator would plausibly tune live — *not* a
per-Stuff/template field default, per-species content, a physics constant, a
safety/recursion guard, a deliberate mechanism cadence, or a value already
gated elsewhere (e.g. `dm`'s recipient cap is enforced by the `dm.yaml`
cardinality, so its controller backstop stays a literal; the introduce
regard bump is a flagged-replaceable demo, so it stays put).

| Key | Seeded value | Read by |
|---|---|---|
| `chat.historyCap` | `200` | `ChannelCatalogue.recordHistory` — per-channel in-memory history ring cap. |
| `chargen.nameMinLength` / `chargen.nameMaxLength` | `2` / `24` | `EnrollController.validateNameToken` — inclusive name-length bounds. |
| `status.maxLength` | `100` | `StatusMixin` `sanitizeStatus` — max rendered status one-liner length. |

Each consumer reads through a local try/catch helper that falls back to the
historical literal when the cache is unread (pre-warm / tests), so the
migration is behavior-preserving and safe on the hydrate-at-boot path.

## Adding a setting

1. Add a `key`/`value` entry to `mud/config/app-settings.yaml`.
2. Add the key constant to `AppSettingKeys` (`lib/config/AppSettings.ts`).
3. Read it where it's consumed: `AppApi.setting(AppSettingKeys.yourKey)`.

That's it — no change to `AppSettings`' persisted shape, no new Api. The
seeder merges the new key into the row on the next boot (existing values
untouched). A setting nobody reads is inert, so the consumer is the real
work.

## Boundaries — what is *not* app settings

- **Per-player** preferences → `EnvironmentMixin` `settings`
  ([shell-environment.md](./shell-environment.md)).
- **Per-Stuff** state → `Propertied` ([properties.md](./properties.md)).
- **Client-UI-state** persisted server-side → its own (TBD) substrate.
- **Secrets / deployment / infra** (Mongo URI, OAuth, ports) → `.env` /
  SSM Parameter Store (deployment.md).
- **Module-internal code constants** → colocated in their owning module.

## Deferred

A richer typed schema (per-setting value types, validation, migration — à
la `SettingsSchemaEntry`); more app-wide knobs (MOTD, world feature flags,
the lounge distribution dials); and the further `AppApi` operations
(`shutdown()` etc.). All land with their own builds; v1 is the two
start-location knobs and the read/write/verb surface.

## History

The build first shipped (`6897f4dd`) with the values in a code constants
map (`lib/config/keys.ts`'s `AppSettingDefaults`) that `AppSettings.loadOrSeed`
copied into the DB, with `AppApi.setting` falling back to the map. MR
review (`b8785c38`) replaced that with the current model: the values live
only in the DB, seeded from `mud/config/app-settings.yaml` by a backend
`AppSettingsSeeder` (a code defaults map was a redundant second on-disk copy
of what the seeder already guarantees). `keys.ts` was deleted (the key
vocabulary folded into `AppSettings.ts`), `loadOrSeed` became the cache-only
`warm`, and the `setting` fallback was dropped.
