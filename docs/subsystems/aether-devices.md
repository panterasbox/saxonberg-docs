# Aether devices — the phone, apps and hosts, the battery, the lock

What lets a person reach past the room: a **tuner** they carry, the **apps**
it runs, the **battery** it runs on, and the **title** that makes it theirs.
Built by the aether-devices build (2026-11); the design is
aether-devices-slate.

## The one job

The aether exists so that chat, DMs and emotes work at a distance, and so
that nobody is ever stranded. A **place carries what is expressed in it**
(arcane-science rule 1): an emote in a room reaches everyone there, NPCs
included, with no device and no sense. Reaching *past* the room needs a
tuner at both ends.

Two promises replace "DMs always work": nobody is stranded without a way
to ask for help, and a social player never meets a mechanic that stops them
talking unless they walk into one. A message that cannot go is **held,
never lost** (below).

## Perceiving, hosting, transmitting — three mixins

| mixin | what it says | composed by |
|---|---|---|
| `AetherMixin` (`lib/message/Aether.ts`) | *this body can perceive the aether once something tunes it* — the gated sense; `_grantsModalities` `verbal-esp`, `emotive-esp` | `Avatar` |
| `AetherHostMixin` (`lib/augmentation/AetherHost.ts`) | *this thing hosts aether software* — the live apps and the persisted installed list | `Avatar` (their wallet, calendar, forums), `Phone`, `AetherImplant` (and so the headset) |
| `CommsMixin` (`lib/comms/Comms.ts`) | *this person talks at a distance, and things wait for them* | `Avatar` only — ⚠ never `Character`: no NPC ever talks at a distance |

`MixinApi.isAetherHost` asks *hosts software*; *tuned* is
`MixinApi.isActive(x, 'AetherMixin')` or the sensorium.

## Tuners — who confers the sense

The acquired conferral leg (`collectAugmentConferralNames`) asks **every
augment the body bears**: slot occupants (a seated implant, a phone in a
hand) and carried ones — a carried device's `environment` verbs sit on its
bearer's recency stack (`CommandGiver.getCommandSources('environment')`),
kept exact on every move, so that stack is the record of what the body
bears. ⚠ A carried augment with no `environment` verb is invisible to the
leg; every device today is an aether host and affords `apps`.

An augment can say **why it is not working** — `conferralRefusal()`:

| reason | when |
|---|---|
| `not-borne` | nobody carries it |
| `off` | switched off (`switch phone off`) |
| `packed` | its bearer holds it `packed` (a rucksack on the back) — `hand` and `ready` tune |
| `dead` | its battery is empty |
| `not-yours` | it is `ownerLocked` and its title is somebody else's |

| tuner | who has it |
|---|---|
| **phone** (`platform/thing/Phone.ts`) | every new player — the starting kit, a bound Hollis Pal in a pocket; sold at the general store |
| **implant** | guests only (a guest's phone would leak into the economy); always on, no battery |
| **headset** | the implant worn; sold at the general store |
| **by nature** | a shade (every shade is the shade species, `AetherMixin` innate) or a born-attuned species; their own host — derived (`isOwnAetherHost()`: the sense active and no borne device conferring it), never declared |

A circle body gets no tuner: the holodeck forces no decision on its user.

## Apps

An app is an `App` (`platform/idea/App.ts`) — an incorporeal `Idea` bound
to its host for life. Rows live under `/platform/idea/app/`: `messages`,
`forums`, `wallet`, `calendar`, `broadcast`. An app's **key** is its row's
last segment.

- `installApp(path)` is the **one** way software gets onto a host: clone the
  row, host it, record it in `installedApps`. Idempotent by path.
- `installedApps` persists; `setInstalledApps` reconciles the live set, so a
  host stands its apps up at birth (template phase) and after a reboot
  (restore phase).
- ⭐ **The app store is documents a pack PUBLISHES**, never a search of
  the template table (stream arrival, 2026-11-24 — the owner: *"template
  paths aren't what make an app an app"*). A pack ships
  `content/apps/<key>.yaml` (kind `app-listing`, `app: <App row>`); the
  listing's leaf is the word `apps install <key>` types, and the kind is
  `keyworded: { unique: true }`, so a second pack listing the same word is
  refused **at install**. `AppStore` (`platform/idea/AppStore.ts`, a
  `DocumentCatalogue`, reached as `TemplateApi.appStore()`) reads them;
  `AetherHostMixin.availableApps()` is async over it. `offersApps` (the
  per-device offer list) is deleted. Shipped listings: `messages`,
  `broadcast`.
- ⭐ **An installed app lends its verbs to whoever bears its device.** An
  app's `static commandContributions` `self` defs are pushed onto the
  bearer's container chain under the `environment` bucket by
  `CommandLogic.lendHostedApps` — one routine beside every `environment`
  push and pop — so `apps install broadcast on phone` gives the phone's
  bearer `program` and `rodney`. When the device will not run (`not-borne`,
  `packed`, `off`, `dead`, `not-yours` — its `conferralRefusal()`), the
  dispatcher passes the command over to another offer of the same verb,
  else refuses with the device's row (`aether-none`, `aether-packed`,
  `aether-off`, `aether-dead`, `aether-not-yours`; `CommandGiver`'s
  `lentDeviceRefusal`).
- **Messages** does nothing by itself: it is what the person's comms looks
  for on the host that tunes them. No Messages, no `dm`.
- The wallet, calendar and forums are installed on the **person**, every
  login, and are not listed by `apps`.

`apps` (`platform/cmd/device/apps.yaml`) — `apps`, `apps install <app>`,
`apps uninstall <app>`, each with an optional `on <device>`. Afforded by the
host through `environment`, so it reaches whoever bears a device; a
person's own host is never in its own container chain.

## Held, never silent

`CommsMixin.sendOrHold(commandText)` is the one question the send verbs
(`dm`, `reply`, a `chat` post, an emote to someone elsewhere) ask. `null`
means go; otherwise the **typed line** is kept in `heldOutbound` (capped at
50) and the refusal names the fix — *a phone does*, *ready it*, *switch it
on*, *charge it*, *that phone runs only for its owner*, *`apps install
messages`*.

Arriving: `Avatar.filterMessage` asks `holdIfUnreachable` **before** the
sense gate, so a frame over the aether (`verbal-esp`, `emotive-esp`) that
cannot be taken now goes to `heldInbound` (capped at 200, overflow counted)
instead of being dropped. Every producer is covered without being touched.

The flush has no event of its own: any frame the person receives, and
`enter`, checks whether something held can move, and delivers *"While you
were away…"* then re-runs the held lines as typed (*"Sending what you held:
`dm bob hi`"*). Both lists ride the person's snapshot.

**Reading needs sight.** A host's `manifestsAs` is `known` (an implant —
straight to the mind) or `read` (a phone's screen). A `read` host holds what
arrives while the person is unconscious or cannot see.

## The battery and the charger

`BatteryMixin` (`lib/supply/Battery.ts`) keeps a `battery` reserve in **Wh**
on the shared `Reserve`. Nothing runs on a timer; it settles when read for
an act:

- **draw** — `standbyW` while on and drawn by a live body (the phone settles
  in its once-a-game-second conferral check; a new drawer starts fresh, so
  an offline body drew nothing);
- **charge** — resting on anything that answers the kernel `Powered` shape:
  `min(acceptW, availablePowerW)` × powered seconds since the stamp; no
  online bound, so a phone on a charger fills while its owner is away;
- **use** — `useWh` per message sent or read through it.

A battery restored from nowhere keeps its stamp; one lifted off a charger
settles across where it was. ⭐ The gap is read off the supply's
**trajectory** at `acceptW`, never off what the meter reads at the visit (the
`Powered` contract), and a battery **never settles from limbo** — it, or what
it rests on, with no container (a room's reset re-seating its charger): it
keeps its stamp and the settle after it lands charges the whole gap. Both were
found by the drive, a phone on a dorm charger that gained nothing while its
owner was away.

The **charger** (`/system/energy/thing/Charger`, row
`/system/energy/thing/charger`) is `GridPowered` plus an `on` surface; it
supplies only while it sits in a place. Duncan Hall's dorms are on the grid
(the University Avenue node) and every dorm room has one. Nobody is billed
for charging. ⚠ No lounge charger: a non-persistent room destroys an owned
good left in it when it is evicted.

`GridPoweredMixin` drops its cached premises on `onMoved`, so a charger
bought at the store is powered by the dorm it is carried to.

## Ownership is the lock

A phone row may be `ownerLocked`: it runs only for whoever holds its title
(`stampedOwner()`); an unstamped one runs for whoever bears it (a circle
cannot stamp a title). A stolen phone is a brick.

A title can be **bound** (`stampChattel(owner, { bound: true })`): the
starter phone's is. `ChattelLogic.transfer` refuses a bound title — the one
chokepoint every transfer passes — and `Stock.buyFrom`, `shelf consign` and
repossession refuse or skip one before any money moves. `title transfer
<good> to <player>` hands on an unbound good's title, with the recipient's
consent.

## Brands

Phone brands are corpos' marks (`BrandedMixin`): **Hollis Pal** (the
starter; sold), **Aevex Meridian** (sold, dear). Vionne's mark, **Vionne
Soir**, is a brand row with no phone yet: a phone row nothing reaches is
content no player can meet (`lint:reachability`), so it waits for lineage
to choose it. Messages carries no brand. Lineage will choose the
starting brand (lineage-slate).

## What the player sees

- `look phone` — on or off, the apps it runs, the charge (*reads 82%*, *is
  dead*), and *bound to you* / *yours* / *not yours*; the brand line.
- The status bar's **AETHER** row (default-pinned) — `aetherState` on the
  `self` card: `tuned 80%`, `tuned`, `packed`, `off`, `dead`, `not yours`,
  `no Messages`, `no tuner`; poked when the word changes.

## Deferred seams

- ⭐ **An app's verbs on its `app-listing` document** (owner, 2026-11-24). `BroadcastApp`
  is a class whose body is only `static commandContributions`, which `lint:data-ideas`
  counts as metadata (its ceiling rose 13 → 14 for it). The fix: the listing declares
  the verbs, `lendHostedApps` reads them from the hosted app's listing, and a
  verbs-only app needs no class — `BroadcastApp` deletes and the ceiling falls back.
- The wallet, calendar and forums as apps on a device; other apps bringing
  verbs (a map, a camera) — `lendHostedApps` is the attach point.
- The app store's economics and publishing — paid apps, who may publish,
  withdrawal, device compatibility — on the `app-listing` document
  (aether-devices-slate).
- An app that WITHDRAWS its verbs while its device cannot work (the owner:
  maybe for some apps, not now) — today every app keeps lending, and the
  device says why not.
- The phone as a held, watchable screen (display-manifestation-slate).
- A power state on the implant — `conferralRefusal()` is where it answers
  (the epoch build).
- A second carried tuner — the signal to lift the tuning rule out of
  `Phone`'s body.
- Modes, the camera, the map app; metered billing for a charger; battery
  wear; blindness as a condition (meets `receptionRefusal` unchanged).
- The lounge charger and arcana's `ManaCharger`, once owned goods survive a
  non-persistent room's eviction.

## History

- **The aether-devices build** (`build/aether-devices`, MR !615, 2026-11-18/19) — waves W1–W10
  as above. Its blocking drive (`packages/wire/tests/aether-devices.dirty.wire.test.ts`, 21 steps)
  took seventeen runs and found thirteen defects no unit test could, among them: authored row
  fields never applied (not declared persistent); a phone that rests on its charger as a
  **placement host**, not a container; the move core stamping the placement after `onMoved`; a
  room's reset re-seating its charger mid-gap (→ never settle from limbo, the gap off the
  trajectory); **every `Switchable` restored in its row's state** (a kernel capture fix, see
  [persistence.md](./persistence.md)); and a glass starter phone that broke when dropped (both
  phones are rubber). The record was the retired plan's § Drive record — `git log
  b8b0fb71d^..07760516f` has each fix. The browser walk is `e2e/tests/aether-devices-walk.spec.ts`;
  the soak drive `packages/wire/tests/aether-devices.soak.wire.test.ts`.
- **Stream arrival** (MR !655, 2026-11-24) — an installed app lends its verbs to its device's bearer
  (`CommandLogic.lendHostedApps` + the dispatcher's pass-over and the `aether-*` refusal rows),
  closing the first deferred seam; the app store became `app-listing` documents a pack publishes
  (`AppStore`, `TemplateApi.appStore()`), `offersApps` deleted; the first lending app is
  **Broadcast** (`program` + `rodney`).
