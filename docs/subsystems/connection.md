# Connection Lifecycle

How a user becomes a connection becomes an avatar — and back out
again. Covers auth, WebSocket bootstrap, the `Login` handoff,
multiplexing, message routing, and disconnect/logout.

This doc is the **plumbing** map. Sibling docs cover related ground
without overlap:

- [state-model.md](./state-model.md) — what `User`/`Avatar` carry as
  data, why there is no `Player` class, the persistent-vs-runtime
  field split.
- [lifecycle.md](./lifecycle.md) — the generic Stuff create/destroy
  choreography (`postRegister`, `onDestruct`).
- [messaging.md](./messaging.md) — Scene composer, Sensor routing,
  MML rendering. This doc points at the boundary; the inside lives
  there.

## The Cast

Five distinct objects participate. Confusing them is the most common
mistake. The dichotomy is laid out in
[state-model.md § The Data Model](./state-model.md#the-data-model);
this is the operational summary.

| Class | Layer | Persisted? | Lifetime |
|---|---|---|---|
| `User` | auth identity | yes (`users` collection) | account |
| `GoogleProfile` | OAuth cache (identity only) | yes (`google_profiles`) | account |
| `TwitchProfile` | OAuth cache + **credentials** | yes (`twitch_profiles`) | per provider link |
| `KickProfile` | OAuth cache + **credentials** (+ the owner's channel slug/broadcaster id) | yes (`kick_profiles`) | per provider link |
| `Avatar` | game-world character | minted identity path (`/platform/agent/Avatar/<playerId>`), snapshot-backed — no per-player template row | from first connection until explicit destruct |
| `Interactive` | live connection | **no** | one WebSocket session |
| `Login` | entry-procedure scratch object | **no** | one entry — destructed when `enter()` finishes |

**`User`** owns a `playerIds: string[]` — the authoritative "which
characters does this user own?" list (`User.ts`). Each id is a
character slot; the matching identity path is `/platform/agent/Avatar/<playerId>`
(minted, snapshot-backed — no per-player template row).

**Providers are co-equal** — every provider gets the full login + link
flow (one unified interface, never a link-only tier). `User` carries
three optional FK fields — `googleProfileId?`, `twitchProfileId?`, and
`kickProfileId?` — with an **at-least-one** invariant
(`User.hasAnyProvider()`, all three counted). A login through any
provider resolves to the same `User`; an authenticated user can **link**
the others and **unlink** any (but not their last one). The spine is
provider-parameterized — `User.profileFieldFor(provider)` is the single
source of truth for the FK field name — rather than a generic
`identities[]` map (premature at N=3). `GoogleProfile` stores **identity
only** (login mints a session and discards the token); `TwitchProfile`
and `KickProfile` additionally persist OAuth **access/refresh tokens**
because the game calls the platform *as the user* for the life of the
link — those token fields are **encrypted at rest** via the
`EncryptedStringMarshaller` (see
[§ Phase 1](#phase-1-http-auth--user-creation) and
[persistence.md](./persistence.md)). `KickProfile` also captures the
owner's **channel** (`slug` + `broadcasterUserId`, best-effort at
auth/link time) — the fields the streaming relay's character-form
resolve and reverse speaker-link read (see
[streaming.md](./streaming.md)). The Kick strategies are the Twitch
transcription with one mechanical delta: **PKCE S256** (`{ state: true,
pkce: true }` on the generic `passport-oauth2` strategy; names `'kick'`
/ `'kick-link'`, routes `/auth/kick[/link]`).

**`Avatar`** is what walks around the world. Composes
`HasInteractiveMixin` so it can carry zero, one, or many connected
`Interactive`s. `user` and `playerId` are runtime-only pointers
stamped from the clone context (`Avatar.ts § postRegister`); the
public surface is `getUser()` / `getPlayerId()`.

**`Interactive`** is a connection. Holds `socketId`, `sessionId`, the
authenticated `User`, and a `holder: HasInteractive & Stuff` that
points at whoever currently owns the connection — `Login` during
entry, `Avatar` during play. The contract is methods-only:
`getSocketId()` / `getSessionId()` / `getUser()` / `getUserId()` /
`getHolder()` / `setHolder()` (`Interactive.ts`). Routing goes
through `ConnectionApi`; nothing dispatches on the holder field
directly.

**`Login`** is a one-shot orchestrator. Composes `HasInteractiveMixin`
too, so the handoff from Login → Avatar uses the same
`interactive.transferTo` mechanism that any future re-handoff would.
Lives only as long as `enter()` takes; `StuffApi.destruct(this)` is
the last line of its body (`Login.ts § enter`).

There is no `Player` class. The id is "still called `playerId`" — see
[state-model.md § No Player class](./state-model.md#no-player-class).

## Login Flow at a Glance

```
HTTP /auth/{google,twitch}
       │
       ▼
Provider OAuth ──▶ /auth/{provider}/callback
                          │
                          ▼
              Backend.handleProviderAuth(provider, …)
                          │   (runRoot frame)
                          ▼
              Application.findOrCreateUserFromProvider(provider, …)
                  ├─ findOrCreateProfile(provider)  (google_profiles | twitch_profiles | kick_profiles)
                  └─ findOrCreateUser(provider, …)  (users)
                       └─ first time? → createDefaultAvatarTemplate
                                         (forks /platform/agent/Avatar/seed →
                                          /platform/agent/Avatar/<new playerId>)
                          │
                          ▼
                  Passport serializes { id, authProvider } into session
                          │
                          ▼
                Client redirected to ${CLIENT_URL}/?auth=success
                          │
        ── client opens WebSocket ──
                          ▼
       httpServer 'upgrade' → sessionMiddleware → userId from session
                          │   (401 if missing)
                          ▼
              Backend.handleWebSocketConnect
                          │   (runRoot frame)
                          ▼
              Application.handleUserConnect
                  ├─ User.findById(userId)
                  ├─ ConnectionManager.createInteractive(...)
                  └─ StuffApi.create(() => new Login(interactive))
                          │
                          ▼
                       login.enter()
                  ├─ interactive.transferTo(login)
                  ├─ PlayerApi.loadAvatarsForUser(user)
                  ├─ interactive.transferTo(avatar)
                  ├─ avatar.teleport(startingRoom, { silent: true })
                  ├─ MessageApi.scene(avatar)…session.link
                  ├─ EventApi.emit(Events.PlayerLoggedIn | …Reconnected, …)
                  └─ StuffApi.destruct(this)
```

## Phase 1: HTTP Auth → `User` Creation

`Server.setupMiddleware()` (`Server.ts:59-110`) wires the middleware
stack: CORS with credentials, JSON/URL parsers, cookie parser,
**`express-session` with `SESSION_SECRET`**, then `passport.initialize()`
and `passport.session()`.

The session middleware instance is **stashed onto the Express app
itself** so the WebSocket upgrade handler can run the same instance
later (`Server.ts:109`). One source of session truth.

```typescript
// Server.ts:81-91
const sessionMiddleware = session({
  secret: sessionSecret,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 24 * 60 * 60 * 1000,    // 24 hours
  },
});
```

`PassportConfig` registers, per provider that has credentials in the
env, a **login** strategy and a **link** strategy:

- **Google** — `passport-google-oauth20`, names `'google'` /
  `'google-link'`.
- **Twitch** — a hand-rolled `passport-oauth2` `OAuth2Strategy` pointed
  at Twitch's endpoints (the verify callback does a Helix `/users` fetch
  to build the profile), names `'twitch'` / `'twitch-link'`. Chosen over
  the lightly-maintained `passport-twitch-*` wrappers. The login scope is
  identity-only (`user:read:email`); chat scopes are deferred.

Each strategy is **gated on its provider's `*_CLIENT_ID/SECRET`
presence**, independent of `AUTH_MODE` — CI / e2e run without those vars,
so the strategy is skipped there and the test-auth seam
(`backend/TestHooks.ts` behind `TestAuthRoutes`) handles login.
All four strategies set **`state: true`** — a session-stored nonce on the
OAuth round-trip that protects both login and (critically) account
**linking** against CSRF.
The login verify callbacks hand off to
`Backend.handleProviderAuth(provider, profile, done)`; the link verify
callbacks to `Backend.handleProviderLink(provider, userId, profile,
done)`.

Routes (`AuthRoutes.ts`):

- `GET /auth/{google,twitch}` — kick off login OAuth.
- `GET /auth/{provider}/callback` — Passport runs verify, then redirects
  to `${CLIENT_URL}/?auth=success` (or `?auth=failure`).
- `GET /auth/{provider}/link` + `/link/callback` — **authenticated**
  (`AuthMiddleware.requireAuth`) account linking.
- `POST /auth/{provider}/unlink` — **authenticated** unlinking.
- `GET /auth/status` — `{ isAuthenticated, user? }`.
- `POST /auth/logout` — `req.logout()` + `req.session.destroy()`.

`Backend.handleProviderAuth` (mirrored by `handleProviderLink` /
`handleProviderUnlink`) wraps the call in a security root frame and
serializes `{ id, authProvider }` into the session:

```typescript
const userId = await ExecutionContextApi.runRoot(
  Backend,
  'findOrCreateUserFromProvider',
  () => app.findOrCreateUserFromProvider(provider, profile)
);
done(null, { id: userId, authProvider: provider });
```

`Application.findOrCreateUserFromProvider(provider, profile)` runs two
upserts:

1. **`findOrCreateProfile(provider, profile)`** — upserts the right
   collection by provider (`google_profiles` keyed on `googleId` /
   `twitch_profiles` keyed on `twitchUserId` / `kick_profiles` keyed on
   `kickUserId`). The Twitch and Kick paths route through the Document's
   `save()` so the `EncryptedStringMarshaller` encrypts the token
   fields — plaintext tokens never reach Mongo.

2. **`findOrCreateUser(provider, profileId)`** — resolves via the
   computed key `User.find({ [User.profileFieldFor(provider)]: id })`.
   If new, constructs a `User` via `await StuffApi.create(() => new
   User())`, sets the right `*ProfileId`, saves, then calls
   the character-mint path when applicable (test-auth provisioning uses
   `Application.createDefaultCharacter` — a seed-overlay clone that
   captures a first persistence-spine snapshot and destructs the
   transient instance; NO per-player template row) and pushes the new
   `playerId` onto `user.playerIds`. In production, characters are
   minted by char-gen's `commit`, not at account creation.

**Linking & unlinking.** `Application.linkProvider(userId, provider,
profile)` upserts the profile, then: attaches it if unowned
(`linked`); no-ops if already on this user (`already-linked`); or
**refuses** if owned by a *different* user (`collision` — no merge).
`Application.unlinkProvider(userId, provider)` clears the FK and
**deletes the orphaned Profile Document** (removing the stored encrypted
tokens), but **refuses removing the user's only provider**
(`only-provider`, defending the at-least-one invariant) and no-ops when
not linked. Both are data-integrity operations on the persistence layer,
not the route handler.

**Token refresh.** `TwitchProfile.applyRefreshedToken(...)` is the
`RefreshingAuthProvider.onRefresh` write-back target: it re-`save()`s the
rotated tokens, which re-encrypts them through the marshaller. (The relay
that *spends* the tokens is a downstream build; this build only stores
them and proves the write-back.)

`createDefaultAvatarTemplate` forks from the seed avatar at
`Avatar.SEED_TEMPLATE_PATH` (`/platform/agent/Avatar/seed`), generates a fresh
`playerId` via `nanoid()`, overlays the user's `name`/`surname`, and
persists the new template via `TemplateApi.saveTemplate`. **If the
seed is missing it throws** — the platform pack must have installed at boot.

Passport serializes `{ id: userId }` into the session
(`PassportConfig.ts § serializeUser`). At this point the persistent state is
done: `users`, `google_profiles`, `domain` (template) all written.
**No runtime objects exist yet.**

## Phase 2: WebSocket Upgrade

`Server.start()` passes the saved `sessionMiddleware` into
`WebSocketService.initialize`.
`WebSocketService` attaches its own handler to the HTTP server's
`upgrade` event:

```typescript
// WebSocketService.ts:51-56
httpServer.on('upgrade', (request, socket, head) => {
  sessionMiddleware(request as any, {} as any, () => {
    this.handleUpgrade(request, socket, head);
  });
});
```

The session middleware runs against the upgrade request **before**
`handleUpgrade` decides whether to accept. `handleUpgrade` reads
`req.session.passport.user.id`; if absent, writes `401` and destroys
the socket. Otherwise, it calls `wss.handleUpgrade` and on success
delegates to `Backend.handleWebSocketConnect(ws, userId, sessionId)`.

**Broadcast principal short-circuit.** Before the session check,
`handleUpgrade` looks for a `?broadcast=<token>` query param (OBS
browser sources can't set WS headers). On a constant-time match
against `BROADCAST_TOKEN`, the upgrade is accepted as the sentinel
`service:broadcast` principal —
`handleWebSocketConnect(ws, 'service:broadcast', …, isBroadcast=true)`
— which routes to the read-only `BroadcastFeed` with **no
`Interactive`** (a pure push target; commands rejected by
construction). See [livestream.md](./livestream.md).

The `WebSocketServer` is configured with `noServer: true` and
`maxPayload: 50MB` (`WebSocketService.ts:44-48`). `ws` does not bind
its own listener; the manual `upgrade` interception is what lets
sessions be checked first.

## Phase 3: `Interactive` and `Login` Bootstrap

`Backend.handleWebSocketConnect` (`Backend.ts:99-134`):

1. Mints `socketId = socket_<timestamp>_<rand>`.
2. Stores the raw `WebSocket` in `socketsBySocketId`.
3. Wires `message`/`close`/`error` handlers.
4. Plants the network → Application root frame and calls
   `Application.handleUserConnect(userId, sessionId, socketId)`.

```typescript
ExecutionContextApi.runRoot(Backend, 'handleUserConnect', () =>
  app.handleUserConnect(userId, sessionId, socketId)
);
```

`Application.handleUserConnect`:

1. `User.findById(userId)` — load the persistent record.
2. `ConnectionManager.get().createInteractive(socketId, sessionId, user)` —
   constructs an `Interactive` via `StuffApi.create` and registers it
   in the `interactivesBySocketId` map.
3. `await StuffApi.create(() => new Login(interactive))`.
4. `await login.enter()`.

`Interactive` is created via `StuffApi.create` rather than `clone` —
it's a runtime-only object built from constructor args, not from a
template (closes over `socketId`/`sessionId`/`user` directly).

The `Login` constructor seeds itself with the Interactive:

```typescript
constructor(interactive: Interactive) {
  super();
  this.interactive = interactive;
  // Login multiplexing never actually happens — it's just consistency
  // with the mixin so MixinApi.isHasInteractive(login) works.
  this.addInteractive(interactive);
}
```

The seed is `addInteractive`, NOT `interactive.transferTo`. The
Interactive's `holder` is still `null` at this point. The first
proper `transfer` happens inside `enter()` (next phase).

## Phase 4: Avatar Materialization and Handoff

The entry procedure splits cleanly across two objects:

- **`Login.enter()`** owns connection routing — take ownership of
  the Interactive, resolve which Avatar this user should connect to,
  transfer the Interactive to the Avatar, hand off to
  `avatar.enter()`, destruct.
- **`Avatar.enter(interactive)`** owns the session-start — anything
  that's "playing as this avatar" (location placement, welcome
  scene, look description, autosave, the PlayerLoggedIn event).

The split keeps each side's concerns aligned with the object whose
lifecycle they belong to. Login doesn't need to know what "starting
a session" means; Avatar doesn't need to know how Login picked it.

### `Login.enter()` — verbatim sequence

1. **`interactive.transferTo(this)`** — formal handoff
   to Login. Calls `interactive.setHolder(login)` and fires
   `onConnectionAttached` on Login.

2. **`PlayerApi.loadAvatarsForUser(interactive.getUser())`** — for
   each `playerId` on the user, returns the existing in-memory
   Avatar if one is registered; otherwise clones from the per-user
   template with context `{ user, playerId }` so
   `Avatar.postRegister` can stamp the runtime fields synchronously.

   ```typescript
   avatar = await StuffApi.clone<Avatar>(
     AvatarClass.getTemplatePath(playerId),
     { user, playerId }
   );
   ```

   **`Avatar.postRegister` registers the avatar with `PlayerApi`
   keyed on `playerId`** — that's what makes "second connection
   finds the same avatar" possible.

3. **`if (avatars.length !== 1) throw`** — multi-character is not
   yet supported. See [§ Not Yet Implemented](#not-yet-implemented).

4. **`interactive.transferTo(avatar)`** — moves the
   holder slot from Login to Avatar. Witness hooks fire (see
   [§ Multiplexing](#multiplexing)).

5. **`await avatar.enter(interactive)`** — hand off to Avatar's
   session-start. Login doesn't introspect the result; whatever
   Avatar needs to do is its responsibility.

6. **`StuffApi.destruct(this)`** — Login is gone. The connection
   has been handed off; nothing depends on Login any more.

### `Avatar.enter(interactive)` — session start

1. **Resolve the starting location.** Consults the avatar's live
   container first — set by the Avatar template's `data.container`
   via Phase 2 `applyContainer` during clone, or by
   `Avatar.restore()` re-hydrating saved state. Only when the
   avatar has no container (a brand-new spawn with no declared
   `data.container`) does the fallback fire:
   `StuffApi.singleton(DEFAULT_STARTING_LOCATION_PATH)` and a
   silent `avatar.teleport(loc, { silent: true })`. Per-Avatar
   persist-back makes this branch the rarer path; the common case
   across restarts is `avatar.getContainer()` already pointing at
   the saved location.

2. **`avatar.startAutoSave()`** installs the periodic persist-back
   timer (see [§ Auto-save lifecycle](#auto-save-lifecycle)).

3. **Welcome scene** at topic `session.link`,
   audience `toSelf`, body `"Welcome back, <fullName>!"`, with a
   **payload** the client needs for bootstrap:

   ```typescript
   .payload({
     userId: interactive.getUserId(),
     socketId: interactive.getSocketId(),
     sessionId: interactive.getSessionId(),
     player: {
       _id: avatar.getPlayerId(),
       honorific: avatar.getHonorific(),
       name: avatar.getName(),
       surname: avatar.getSurname(),
       nameSuffix: avatar.getNameSuffix(),
       alternateNames: avatar.getAlternateNames(),
       pronouns: avatar.getPronouns(),
     },
   })
   ```

4. **`this.autoLookOnArrival()`** — delegates to `MobileMixin`'s
   existing auto-look path: forces the `look` verb (with focus
   reset), same code that fires after a traversal. Avoids
   reimplementing the look output in Avatar.

5. **`EventApi.emit(Events.PlayerLoggedIn | Events.PlayerReconnected,
   { playerId, userId })`** — engine event for any observer (audit,
   achievements, the social presence relay). A first-ever `enter()` for
   this instance is a fresh login (`PlayerLoggedIn`); a second `enter()`
   (a connection returning to a body that lingered linkdead) is a
   reconnect (`PlayerReconnected`), discriminated by a transient
   `sessionActive` flag that survives the linkdead window. (Separately,
   `Application.handleUserConnect` captures the connection's country of
   origin via `interactive.recordOrigin` when the Interactive is created
   — see [social-graph.md](./social-graph.md) § Country of origin.)

After `Avatar.enter()` returns, the user is in-world.
`interactive.getHolder() === avatar`; Login is destructed and
unregistered.

## Multiplexing

> Multiplexing was an explicit MUST-NOT-regress requirement
> (`state-model.md:243-249`). It works as follows.

`HasInteractiveMixin` (`HasInteractive.ts`) gives any composing
class:

```typescript
protected interactives: Set<Interactive>   // host-internal storage
addInteractive(i)                          // primitive
removeInteractive(i)                       // primitive
clearInteractives()                        // bulk drop (destruct-time)
getInteractives(): ReadonlySet<...>        // safe read
isConnected() / isLinkdead()               // count-based predicates

// Persisted client UI state — see "Client state" below.
_clientState: Record<string, unknown>      // persistent field
getClientState<T>(key)                     // stored value, else schema default
setClientState(key, value)                 // schema-validated write
snapshotClientState(): Record<...>         // dense snapshot for welcome payload
```

External code does NOT call `addInteractive`/`removeInteractive`
directly; `interactive.transferTo` and `interactive.detach` are
the only sanctioned mutators (`connection.ts:80-150`).

### A second connection for the same user

`PlayerApi.loadAvatarsForUser` checks the registry first
(`PlayerApi.ts:102`). If the Avatar is already registered (because
a previous connection cloned it), the new Login receives the
**existing** Avatar object. `newInteractive.transferTo(
existingAvatar)` then adds the new connection to the same Set.

The Avatar now has two Interactives. `Avatar.handleMessage` iterates
the whole Set:

```typescript
protected override handleMessage(frame: MessageFrame): void {
  const app = Avatar.getApplicationInstance();
  for (const interactive of this.interactives) {
    app.sendMessageToInteractive(interactive, frame);
  }
}
```

Every message the avatar perceives is sent to **every** connected
client. Multiplexed sessions share a single in-world identity but
each get their own outbound stream.

### Witness Hooks

`interactive.transferTo` fires per-connection AND presence-transition
hooks (`connection.ts § transfer`):

| Hook | When | Why |
|---|---|---|
| `onConnectionAttached(conn)` | every transfer | per-device events |
| `onConnectionDetached()` | every detach | per-device events |
| `onLinkdead()` | count crosses 1 → 0 | last connection dropped |
| `onLinkRestored()` | count crosses 0 → 1 | first connection back |

All four are **optional**. `Avatar` overrides `onLinkdead` to emit a
global presence event, split by intent: `Events.PlayerLoggedOut` for a
**deliberate** leave (sign-out / switch-character — the client closed
with `INTENTIONAL_LEAVE_CLOSE_CODE` 4000, which
`Application.handleUserDisconnect` recorded via `Avatar.setLeaveIntent`),
else `Events.PlayerDisconnected` for an **involuntary** linkdead drop
(the body lingers; the next `enter()` is a reconnect). Per-device hooks
aren't currently used.

[Metabolism](./metabolism.md) is a **read-only** consumer of presence and
adds **no connection-layer work**: its lazy reconcile checks
`isHasInteractive(self) && isLinkdead()` to freeze the in-session
metabolic clock (a linkdead body lingers in-world but doesn't tick), and
logout state rides the existing `Avatar` save/restore (a far-past-gap
guard drops the away time on relog). There is no `logout` command, no
sleep flag, and no voluntary-vs-involuntary distinction — both absences
just stop the clock.

### Client state

`HasInteractiveMixin` also owns the **persistent UI state**
surface: every HasInteractive-bearing thing has, by definition, a
client attached, and that client has settings worth keeping across
sessions (tabbed-terminal layout, theme, notification prefs,
keybinds, channel mutes, saved MQL queries, onboarding flags).
Putting the storage on the same mixin pairs the two concerns —
"I hold connections" and "I persist state for whoever's on the
other end."

**Storage.** `_clientState: Record<string, unknown>` —
Hydrator-saved. Keys are dotted strings; values are JSON-shape per
the schema.

**Schema.** `static clientStateSchema: ClientStateSchemaEntry[]`
on `HasInteractiveMixin` itself — one flat array. Each entry
declares `{ key, defaultValue, description?, validator? }`.
Today's entries (`console.tabs`, `console.activeTab`, and
`style.overlay` from the message-rendering build); future features
append to the same array. *No* prototype-chain walker: the
schema's scope is exactly HasInteractive-bearers, so there's no
useful distinction between substrate keys and feature-mixin keys.
If the array grows past comfortable, externalize (YAML / DB /
per-feature mixin registry) — not before.

**Methods.** `getClientState<T>(key)` returns the stored value or
the schema default; `setClientState(key, value)` validates +
writes; `snapshotClientState()` returns a dense map (every
declared key with its stored-or-default value);
`pushClientStateUpdate(key, value)` pushes the authoritative
value to every connected Interactive.

**Wire surface.** Three flows:

- **Server → client at session-establish.**
  `ConnectionEstablishedPayload.clientState` carries
  `avatar.snapshotClientState()` in the welcome scene.
- **Client → server on UI mutation.**
  `{ type: 'client-state-write', payload: { key, value } }` —
  generic inbound message. The handler calls
  `avatar.setClientState(key, value)` (schema check happens
  there) then `avatar.save()`.
- **Server → client on server-initiated mutation.**
  `{ type: 'client-state-update', payload: { key, value } }` —
  outbound push parallel to `client-state-write`. Server code
  that mutates a key out-of-band (the `cockpit` verb is the v1
  caller) calls `host.pushClientStateUpdate(key, value)` so the
  client re-renders without waiting on the next reconnect
  snapshot. The implementation is strategy-injected
  (`setClientStateUpdatePush` in `HasInteractive.ts`) to break
  the `Application → Avatar/Login → HasInteractive → Application`
  load-time cycle that direct import would create.

One generic wire trio covers every feature. Adding a new key =
append one schema entry. No feature-specific wire messages, no
per-feature persistence shim.

**Contrast with adjacent stores.**

| Substrate | Purpose | Where composed |
|---|---|---|
| `EnvironmentMixin` (settings) | Player-tunable knobs via the `settings` shell command | Avatar + a few others |
| `PropertiedMixin` | Universal per-Stuff key/value bag | Every Stuff in the world |
| `HasInteractiveMixin._clientState` | UI state the server holds on behalf of a client | Only where a client attaches |

See [shell-environment.md](./shell-environment.md) for settings;
[properties.md](./properties.md) for PropertiedMixin.

The transition fire-once-per-edge semantics matter: a user with two
devices who closes one tab does NOT trigger `onLinkdead`. Only the
last close does.

### Inbound from a multiplexed session

A multiplexed pair is symmetric for OUTBOUND but asymmetric for
INBOUND: each socket sends commands independently and the dispatcher
treats them as if the same avatar typed both. There is no input
arbitration today. (Two devices typing `attack` simultaneously will
each enqueue a command.)

## Inbound Messages (Client → Game)

```
ws 'message' → Backend.handleWebSocketMessage(socketId, data)
                      │   (runRoot frame)
                      ▼
              Application.processUserMessage(socketId, message)
                      │
                      ▼ switch on message.type
              ┌───────┼───────┐
            echo    ping    command
              │       │       │
              ▼       ▼       ▼
            echo    pong   handleCommand
                              │
                              ▼
                  giver.executeCommand(text, ctx)
```

`Backend.handleWebSocketMessage` parses JSON and dispatches under
another root frame. Bad JSON sends back an `error` frame.

### ⭐ Two inbound lanes per socket

A client's messages are **serialized per socket** so an actor's
commands process in arrival order, never interleaved. (Without this,
two rapid commands that clone the same controller template would trip
`StuffApi.clone`'s in-flight cycle guard — see
[char-gen.md](./char-gen.md).) `Application.processUserMessage` is
`async` and awaits its handler, so a lane spans the full dispatch.

⚠ That is exactly why there are **two** lanes. A command that raises a
prompt does not settle until the prompt is answered, so putting the
ANSWER in the same lane deadlocks the socket: `prompt-response` waits
for the command, which waits for `prompt-response`. Every interactive
prompt raised from inside a command dispatch was unanswerable — a
well-formed frame, no error, nothing logged, and a prompt that hung
forever.

`prompt-response` and `prompt-cancel` therefore ride a **second lane**.
They are replies addressed by `promptId` rather than by position — an
interrupt, which is what a prompt is — so they keep order against each
other while never waiting on a command. Bypassing sequencing
altogether would have fixed the deadlock too, and would have let a
reply interleave with an unrelated queued command; a second lane costs
one Map and gives that up for nothing.

**Who owns what:** `ConnectionApi.sequenceInbound` decides *which lane*
(module-scope state in `ConnectionLogic`, the `PromptLogic` registry
precedent, so an in-flight ordering guarantee survives the singleton's
dest/recreate). `Backend` supplies *what frame it runs in* — the root
frame and the socket's circle scope. That split is not taste:
`ExecutionContextApi`'s frame mutators are gated to boundary files
(`backend/**`, `mud/api/**`, security, `CommandGiver`), a logic
singleton is not a boundary, and an inbound turn genuinely begins at
the transport.

> ⚠ The `--async` interaction, because it is not obvious. The async
> detach happens at `_executeOne`, *after* everything accept-time — so
> a prompt raised in a controller BODY under `async: true` was never in
> a lane and always worked, while an **accept-time** prompt (the
> `confirm-prompt` phase, MQL disambiguation) sat in the lane and
> deadlocked regardless of mode. `--async` was an accidental workaround
> for half the bug. A verb carrying `async: true` because "prompts
> hang" should be re-examined on its own merits.

`Application.processUserMessage` looks up the Interactive by
socketId, then switches on `message.type`:

- `echo` — round-trip the payload (debug).
- `ping` — reply `pong` with timestamp.
- `command` — routed to `handleCommand` (`backend/inbound/command.ts`).

`handleCommand` requires `interactive.getHolder()` to be a
`CommandGiver` — checked by capability via `MixinApi.isCommandGiver`,
never `instanceof` — and anything else gets a `No active character`
error. **Both `Avatar` (in-world) and `Login` (pre-world char-gen /
roster) are `CommandGiver`s**, so the same path serves both; the
char-gen `enroll`/`play` verbs ride this exact pipeline (see
[char-gen.md](./char-gen.md)). The placeless guard (holder has no
container) is **Avatar-only** — a `Login` is intentionally locationless
and still dispatches. The handler builds the `CommandContext` and calls
`giver.executeCommand(text, ...)` (`giver` being whichever
`CommandGiver` holds the connection). The result is discarded — prose
is fired via `Scene` inside the controller; success/failure reaches the
client via the dispatch-response envelope (see
[command-routing.md](./command-routing.md)).

## Outbound Messages (Game → Client)

The full pipeline (Scene composer, audience routing, MML, sensors)
lives in [messaging.md](./messaging.md). The connection-relevant
boundary:

```
MessageApi.scene(actor)…send()
              │
              ▼
recipient.onMessage(frame)         (SensorMixin chokepoint)
              │
              ▼
recipient.handleMessage(frame)     (after filterMessage)
              │
              ▼ (Avatar override)
for (const i of this.interactives):
    i.sendMessage(frame)
              │
              ▼
Application.sendMessageToInteractive(i, frame)
              │
              ▼
Backend.sendMessageToSocket(socketId, frame)
              │
              ▼
ws.send(JSON.stringify(frame))     (only if ws.readyState === OPEN)
```

`interactive.sendMessage` / `sendEnvelope` are the mudlib's **only**
way onto the wire; they forward through `ConnectionLogic` to
`Application`, which owns the Backend reference and stamps the
per-Interactive `frameId`. A `Stuff` never holds an `Application`
reference — under [the import boundary](../architecture.md) a mudlib
module may not import `backend/` at all, so the connection subsystem
owning the socket *and* the send is what makes the layering hold.
Avatar's override at `handleMessage` is what implements multiplexing.
`Backend.sendMessageToSocket` is the final network call; logs and
skips if the socket is missing or closed.

## Disconnect

WebSocket close is the canonical disconnect. The browser navigating
away, the user closing the tab, network failure, and explicit client
close all converge on `ws.on('close', code)`. The **close code**
distinguishes a deliberate leave from an involuntary drop: the client's
`disconnect()` (sign-out / leave-world) closes with
`INTENTIONAL_LEAVE_CLOSE_CODE` (4000); a tab-close / network failure
carries a normal/abnormal code. `handleWebSocketClose` reads it and
passes `intentional` down, so the presence layer can fire `loggedOut`
(deliberate) vs `disconnected` (linkdead) — see the hooks table above.

```
ws 'close' (code) → Backend.handleWebSocketClose(socketId, code)
                       │
                       ├─ socketsBySocketId.delete(socketId)
                       │   (runRoot frame; intentional = code === 4000)
                       ▼
              Application.handleUserDisconnect(socketId, intentional)
                       │  └─ if intentional: holder.setLeaveIntent(true)
                       │
                       ├─ interactive.teardownSubstrateState()
                       │     (cancel MQL + forum subs, reaction
                       │      streams, reject pending prompts) ← before remove
                       ▼
              ConnectionManager.removeInteractive(socketId)
                       │
                       ├─ StuffApi.destruct(interactive)
                       │     │
                       │     ├─ interactive.onDestruct()
                       │     │     └─ this.detach()
                       │     │            ├─ avatar.removeInteractive(this)
                       │     │            ├─ interactive.holder = null
                       │     │            ├─ avatar.onConnectionDetached?()
                       │     │            └─ avatar.onLinkdead?()  ← if last
                       │     ├─ shadow detach
                       │     └─ destroy() → unregister
                       │
                       └─ interactivesBySocketId.delete(socketId)
```

### Per-Interactive substrate teardown

Before the Interactive is removed, `handleUserDisconnect` calls
`interactive.teardownSubstrateState()`. The per-Interactive teardown
list — cancel live subscriptions (MQL **and** forum), cancel reaction
streams, reject pending prompts (`'host-disconnected'`) — **lives on
`Interactive` itself**, not enumerated at the network boundary.
Application just triggers it. It runs *before* `removeInteractive` so
any final substrate-side delivery still has a live Interactive to
address (and a controller's prompt-catch block can react while the
Interactive is still around). Each subsystem exposes its own
`cancelAllForInteractive(this)` (`MqlSubscriptionApi`, `ForumsApi`,
its own reaction/prompt teardown; adding a new per-Interactive
substrate means appending one call in `teardownSubstrateState`, not
touching Application.

### What survives the disconnect

The Interactive is destructed and gone. **The Avatar is not.**

Nothing in the disconnect path destructs the Avatar. It stays
registered with `PlayerApi` (and in `StuffApi.objectsById`)
indefinitely, its location intact, ready for the user to reconnect
and have `loadAvatarsForUser` find the same in-memory instance.

That's load-bearing — it preserves world state across reconnects and
is the foundation of multiplexing — but it also means the registry
**grows monotonically** until something explicit destructs the
Avatar. There is currently no idle eviction (see
[lifecycle.md § Open Design — Idle Eviction](./lifecycle.md#open-design--idle-eviction)).

`Avatar.onLinkdead` emits `Events.PlayerLoggedOut`. Listeners can
react (audit, "the adventurer fades from view" perception broadcasts
in the future, etc.).

### Avatar.onDestruct

When something DOES destruct an Avatar (test cleanup, future
character-deletion flow), `Avatar.onDestruct` fires a final
fire-and-forget `Avatar.save()`, cancels the periodic persist-back
timer, unregisters from `PlayerApi`, and detaches every still-
connected Interactive via `interactive.detach`. The detach loop
guarantees no Interactive is left holding `holder = avatar` after
the avatar is gone, so the witness pipeline stays consistent even
if a caller didn't drop the connections first.

### Auto-save lifecycle

`Avatar.startAutoSave()` is invoked from `Avatar.enter()` during
session start. It installs a `ScheduleApi.recurring` timer that
calls `avatar.save()` (which captures through the self-persistence spine —
`PersistableApi.capture` → `holder_snapshots`; see
[persistence.md](./persistence.md#the-self-persistence-spine-persistable)).
Three knobs are set explicitly:

- **Cadence**: resolved from the `world.autosave.interval` setting
  (declared on `Avatar`, default 5 minutes). Per-Avatar overrides
  fall out of the standard `resolveSetting` lookup chain
  (persistent store → schema default). The interval is resolved
  **once** at install time; mid-session changes to the setting do
  not restart the running timer (re-login picks up the new
  cadence). Documented limitation.
- **`propagateAttribution: false`** — the periodic save isn't
  causally a follow-on of the login command; the attribution chain
  is severed so callback frames carry no `causingCommandId`.
- **`mode: 'fixed-delay'`** — drift-tolerant; cadence is "roughly
  every N milliseconds," not a guaranteed wall-clock rate. Prevents
  pile-ups if a save runs long.

`stopAutoSave()` cancels the handle via `ScheduleApi.cancel`. It's
called from `Avatar.onDestruct` before the existing detach loop.

**Concurrent saves are acceptable.** The periodic timer, the
linkdead-driven `onDestruct` save, and any manual `eval avatar.save()`
can run in parallel; each produces a valid full-state snapshot. The
correctness invariant — synchronous capture before the first await —
lives in `PersistableApi.capture`'s implementation; MongoDB's
`replaceOne` resolves write ordering as last-write-wins.

**Multiplexed-session restore.** `Avatar.restore()` is a
developer/admin operation. v1 does NOT add multi-connection
synchronization; connections that observe field flips during a
restore see inconsistent state. Documented limitation. When the
gameplay surface for restore lands (e.g., a "rewind" verb),
coordination will likely shape into a brief `DurativeActivity` that
blocks input for the duration.

**HMR caveat.** A hot-reload of `Avatar.ts` does NOT re-install the
periodic timer on existing instances (their prototype chains keep
the old class). The captured callback closure still works
(field-name dispatch); but admins reloading Avatar.ts in production
should `destruct /platform/agent/Avatar/<playerId>` to clear stale handles and
let the next reconnect pick up the new class.

## Logout (HTTP)

`POST /auth/logout` (`AuthRoutes.ts:67-84`) calls `req.logout()` then
`req.session.destroy()`. The session cookie is invalidated; any
subsequent attempt to upgrade a WebSocket on it will fail at
`WebSocketService.ts:80` with 401.

**Logout does NOT close existing WebSockets.** The route is
HTTP-only. If the client wants to drop in-flight game sessions on
logout, the client must close the socket. If it doesn't, the socket
stays open until the network or browser closes it; on close, the
disconnect path runs as normal. The session being destroyed doesn't
break the live socket because the socket no longer goes through the
session middleware after upgrade — only re-upgrades do.

There is no broadcast "the user has logged out" frame from the
server. (Disconnect doesn't send one either; the WebSocket close is
the signal.)

## Security Frames at the Network Boundary

Every entry from the network into Application is wrapped in
`ExecutionContextApi.runRoot(Backend, '<method>', () => …)`:

| Site | Backend method | Application method |
|---|---|---|
| WebSocket connect | `handleWebSocketConnect` | `handleUserConnect` |
| WebSocket message | `handleWebSocketMessage` | `processUserMessage` |
| WebSocket close | `handleWebSocketClose` | `handleUserDisconnect` |
| OAuth login callback | `handleProviderAuth` | `findOrCreateUserFromProvider` |
| OAuth link callback | `handleProviderLink` | `linkProvider` |
| Unlink (POST) | `handleProviderUnlink` | `unlinkProvider` |

These root frames give the call-stack a well-defined bottom: when a
mudlib method later checks `ExecutionContext`, it sees `Backend` as
the caller at the network boundary, not `null`. Application itself
is annotated `@CallSecurity(SecurityPolicies.Public)`; the comment
there spells out that the
decorator is forward-compatible declaration of intent — instance
methods on Application aren't proxy-mediated. The runtime gate
lives further down, on the Apis Application calls into.

See [call-security.md](./call-security.md) for the broader policy
taxonomy and how `FrameKind`/`runRoot` plant frames.

## Things This Subsystem MUST NOT Regress

- **Multiplexing.** Multiple Interactives → one Avatar; same
  in-memory instance across reconnects (`PlayerApi.loadAvatarsForUser`
  reuses); messages broadcast to every connected device
  (`Avatar.handleMessage`).
- **Session-validated WebSocket upgrade.** No userId in session →
  401 + destroy. The same `sessionMiddleware` instance is shared
  with HTTP; do not duplicate session config in two places.
- **Login destructs itself.** It exists for the entry procedure
  only. Don't keep references to it past `enter()`. Don't subclass
  it for "kept-alive entry contexts" — that's the Avatar's job.
- **`interactive.transferTo` / `detach` are the only mutators of
  the holder slot.** Calling `interactive.setHolder(...)` directly
  skips witness hooks and the cross-cutting
  `Events.ConnectionAttached` event. Don't call
  `addInteractive`/`removeInteractive` directly from outside the
  Api/mixin layer either — those are primitives the Api orchestrates
  in the right order.
- **Network → Application root frames.** Every entry from Backend
  goes through `ExecutionContextApi.runRoot(Backend, ...)`. Adding a
  fifth entry path (e.g. a REST command endpoint) requires planting
  the same frame.
- **Avatar registration via `postRegister`, not constructor.** The
  context bag (`{ user, playerId }`) reaches the Avatar there, and
  registration must happen after the Stuff is in `objectsById` —
  see [lifecycle.md](./lifecycle.md).
- **Avatar persists across disconnect.** The current behavior
  (Avatar stays registered after the last Interactive drops) is
  load-bearing for reconnection. Eviction must run the same cleanup
  path as explicit `StuffApi.destruct` — see
  [lifecycle.md § Open Design — Idle Eviction](./lifecycle.md#open-design--idle-eviction).

## Not Yet Implemented

- **Multi-character selection.** `Login.enter` throws if the user
  has zero or more than one playerId. The character
  list / pick-a-character UI is the missing piece. When it lands,
  the `Login` body splits into "load avatars → present picker" then
  "transfer to chosen avatar" — the `interactive.transferTo` machinery
  is already general enough to support it.
- **WebSocket close on logout.** `POST /auth/logout` invalidates the
  session but leaves the WebSocket open. Either the route should
  enumerate the user's live sockets and close them, or the client
  should close before calling logout.
- **Avatar idle eviction.** Avatars stay registered indefinitely
  after the last disconnect. See
  [lifecycle.md § Open Design — Idle Eviction](./lifecycle.md#open-design--idle-eviction).
- **Persist-back of Avatar runtime state.** Avatar mutations are
  in-memory only; the avatar template is read on first clone and
  never written back. See the comment on `Avatar.onDestruct`.
  Tracked under the
  unified-model "persist direction" in
  [state-model.md § Not Yet Implemented](./state-model.md#not-yet-implemented).
- **Distributed / multi-process Interactive tracking.** `ConnectionManager`
  and `Backend` are single-process singletons. There is no shared
  state for cross-instance routing.
- **Disconnect / system frames to the client.** No
  `session.link` or "you have been disconnected" frame is
  sent on close — the client only sees the WebSocket close event.
  No `session.link` variant either; an upgrade attempt on a
  destroyed session just gets 401.
- **Inbound arbitration for multiplexed sessions.** Two devices typing
  simultaneously each enqueue a command. No single-active-input
  policy.

## Cross-References

- [state-model.md](./state-model.md) — `User`/`Avatar` data model,
  no `Player` class, what's persistent vs runtime
- [lifecycle.md](./lifecycle.md) — generic Stuff create/destroy,
  `postRegister`, `onDestruct`, idle eviction (open design)
- [messaging.md](./messaging.md) — Scene composer, Sensor routing,
  audiences, MML rendering (the "inside" of outbound delivery)
- [command-routing.md](./command-routing.md) — command pipeline reached via
  `avatar.executeCommand` from `processUserMessage`
- [call-security.md](./call-security.md) — `ExecutionContextApi.runRoot`,
  `FrameKind`, policy taxonomy
- [persistence.md](./persistence.md) — the `Document` track that
  `User`, `GoogleProfile`, and `TwitchProfile` ride on; the
  `EncryptedStringMarshaller` (token fields encrypted at rest) rides the
  `fieldMeta`'s marshaller entries seam there
- broadcast-patronage-track.md
  — this multi-provider build is Phase 1 of the go-live track (the
  keystone the Twitch chat relay and capital→stake ledger depend on)

## History

- **Multi-provider auth (auth-providers build, 2026-06).** The
  Google-only spine was generalized to be provider-parameterized and
  Twitch added as a co-equal login provider: `handleAuthenticationSuccess`
  → `handleProviderAuth` (+ `handleProviderLink`/`handleProviderUnlink`),
  `findOrCreateUserFromGoogle` → `findOrCreateUserFromProvider`, `User`
  FKs made optional with an at-least-one invariant, `TwitchProfile`
  (credential-bearing, token fields encrypted via
  `EncryptedStringMarshaller`), account link/unlink routes, and
  `session.authProvider`. Deferred: chat scopes, account merge,
  provider-side token revocation, name-refraction, YouTube. Seeding slate:
  auth-providers-slate.md.

- **Disconnect teardown moved onto `Interactive` (forums build,
  2026-06).** The per-Interactive substrate teardown (cancel MQL + forum
  subscriptions, reaction streams, reject pending prompts) was lifted off
  `Application.handleUserDisconnect`, where the cancellations were
  enumerated inline, onto a new `Interactive.teardownSubstrateState()`.
  Application now just triggers it (before removing the Interactive). The
  forums build was the trigger — adding forum-subscription teardown to the
  list made the case for one home on the Interactive that each subsystem
  appends to via its own `cancelAllForInteractive`. See
  [forums.md](./forums.md).

## Parked bodies (sandbox)

While a player is inside a sandbox circle their field avatar is
**parked** — it keeps the `PlayerApi` registry slot, holds no sockets,
and the player is wearing a vessel elsewhere. Three consequences land
in this subsystem:

- **`Avatar.isConnected()` follows the live vessel while parked.**
  Sockets live on the vessel, so the inherited "any Interactives?"
  answer is `false`, and every consumer of it — `who`, the `online`
  scope that `dm`/`tell` resolve against, the presence roster, notify —
  reads that as *offline*. Stepping into your own circle made you
  unreachable to the whole world.
- **The registry slot is guarded by object identity.**
  `PlayerApi.unregisterAvatar` deletes only when the avatar holding the
  slot IS the one being unregistered. A playerId no longer implies a
  unique body: a wire body reports the REAL playerId (the identity
  thread), so deleting by id alone meant every vessel reaped evicted
  its player's actual body from the registry — and the next connection
  then materialized a second one and collided on the persistence spine.
- **Transport writes cross the boundary INBOUND only.** The lifecycle's
  reads are symmetric; its writes (`setHolder`, `add`/`removeInteractive`,
  `onConnectionAttached`/`Detached`, `onLinkdead`, `onLinkRestored`) are
  exempt only when the context is field and the receiver is
  circle-scoped — the engine reaching into a circle, never circle code
  reaching out. These are `@hook`s, so they are structurally
  ungateable, and `Avatar.onLinkdead` has a real body.

See [sandbox.md](./sandbox.md).
