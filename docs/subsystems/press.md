# Press (publishers, releases, the news ticker)

**What publishers put out, and who gets to read it.** A *release* is one
item — a code change, an operator decision, a scheduled downtime, an
in-world announcement — published **by an organization**, carried live on
the news ticker to everyone signed in, and (when the operator lists the
publisher) readable on the start screen by somebody who has never signed
in at all.

Deliberately **not** the forums ([forums.md](./forums.md), a two-way
*deliberation* substrate): a release is one-directional, low-interaction,
chronological — closer in spirit to the OOC Twitch relay
([streaming.md](./streaming.md)) and the presence relay's
`self.group` frame ([social-graph.md](./social-graph.md)).

Source of truth for this subsystem.

> **Renamed from `bulletin.md`.** The `bulletins` collection retired when
> releases moved into the document tree, and a `Bulletin` class standing
> over a store that no longer exists is exactly the half-migration that
> misleads the next reader — so the vocabulary went with it. The rename
> inventory is at the foot of this doc.

---

## ⭐ The three decisions worth knowing before reading the code

### 1. A publisher is an organization

Publishing is a **hat an organization wears**, beside the Business hat
that trades. `PublisherMixin` requires `OrganizationMixin` on its base, so
a publisher always has a chart — positions, holders, an appointing
authority ([employment.md](./employment.md)). A newspaper, when one
exists, is an organization that trades *and* publishes: one entity, both
mixins.

### 2. The appointing authority appoints. The position publishes.

```
mayPublishAs(principal, org) = holdsPublishingPosition(principal, org)
```

Full stop. **Holding an organization's appointing authority is not an
alternate route to publishing through it.** A President has no
press-secretary powers; they appoint a press secretary. An earlier draft
had `holdsAuthority(...) OR holdsPublishingPosition(...)`, which collapsed
two different powers into one — the explicit negative (a committee member
holding no publishing position is **refused**) is the first test in the
publisher suite for that reason.

⭐ This *strengthens* the founder doctrine rather than straining it. The
founder's specialness lives entirely at the **appointment** step — the
pool-of-one backstop inside `isCommitteeMember` lets them fill the
position — and publishing is earned the way anyone earns it. **Nothing in
the publish path consults the founder at all.**

Its honest consequence: **an organization with no comms director publishes
nothing**, and the founder's first act is to appoint one (possibly
themselves). That is correct behaviour, and it is the gazette slate's own
*"a locality with no gazette is a fact about that locality."*

### 3. A release lives in the document tree, not a system collection

The sort rule (legal-code-slate):
*collections cut across jurisdictions and are queried by system; the tree
is place / owner / division of labor — **would you query it across all
jurisdictions?*** Releases fail that cleanly. **You read a publisher's
feed.** Nobody queries releases globally — even the front page reads a
short enumerated list of publishers, each by prefix. They are owner-scoped
content with a place: a municipality's press releases are the
municipality's.

So a release is a `StoredDocument` (`kind: 'release'`) at
`<feedPath>/<id>` ([document-store.md](./document-store.md)), **owned by
the publisher organization** rather than by the person who typed it.

---

## `PublisherMixin`

`lib/press/Publisher.ts`. Composed at the concrete class, over
`OrganizationMixin` (the `MountableMixin`-requires-`Slotted` idiom — the
base constraint makes applying it to a non-organization a compile error).

| field | what it declares |
|---|---|
| `label` | the masthead a reader sees; falls back to the org path so a byline is never blank |
| `realm` | `ooc` (operator) vs `world` (in-fiction). ⚠ **A release derives this** and cannot supply one |
| `visibility` | the default reach — `public` \| `members`. A release may narrow it, never widen it |
| `feedPath` | the document-tree branch its releases live under |
| `publishingPositions` | which positions may publish. **Empty = any position**, the small-shop case — not a wildcard for outsiders |

⚠ **`ReleaseVisibility` is ordered most-public-first**
(`['public', 'members']`) and that order is **load-bearing**: the clamp is
a `max` over the ordinal, so an inverted list serves members-only releases
publicly. ⚠ An unauthored or misspelled `visibility` coerces to the
**narrow** end.

`obj/Organization.ts` is the concrete instanceable organization — a
ministry, a press office, a registry — composing
`PublisherMixin(OrganizationMixin(PostRegistrationMixin(Idea)))`. It is
what a template's `class:` names; **nothing instances `/lib/`**.

## The entitlement — `EmploymentApi.mayPublishAs`

Exactly *does the principal hold a non-exited position at this
organization whose key is in `publishingPositions`* (empty ⇒ any
position). It lives on `EmploymentApi` rather than the press face because
it reads the **one holder-resolution path**
(`holdersByPositionImpl`, [employment.md](./employment.md)) — routing it
through a press logic singleton would mean either a thin wrapper or a
cross-logic import, both banned. That shared path is also why the
quit/fired suppression exists **once**: an explicit exit is never
resurrected by the authored roster, here or anywhere.

Fails closed on an organization that does not publish, one whose path does
not resolve, and a principal with no durable identity.

## `Release`

`lib/press/Release.ts` — a **value object** over the stored document's
`data`, not a `Document` of its own.

Stored: `releaseId`, `kind`, MML `headline` + `body`, `author` (the acting
author's durable `templatePath` — a stored identity string, **not** an
`AuthoringEvent`), `publishedAt` / `expiresAt` (**epoch-ms numbers, not
`Date`**; `0` = never), `pinned`, `retracted` (a **soft** delete — the
document stays, the window drops it), the optional `visibility` narrowing
(`null` = inherit) and `source` (a `repost`'s provenance line).

**Derived, never stored:**

- `getPublisher()` — the document's `owner`.
- `getRealm()` — the publisher's realm. ⚠ One source, so it cannot drift,
  and nobody can claim to speak in-fiction on an operator's feed.
- `getVisibility()` —
  `max_restrictive(publisher.visibility, own ?? publisher.visibility)`.
  A max over a two-value ordinal, so it is total and monotone and **the
  non-widening direction is a consequence of the shape** rather than a
  check that can be forgotten. Applied on *read*, so a publisher that
  later opens up never retroactively widens a release that asked to be
  narrow — and one that narrows re-clamps everything it has published.

⚠ **`Release.publishedByOwner(release)`** — *is this actually a release?*
The tree is a **shared store**, so `kind: 'release'` is a tag anyone who
can write a document can apply. Every read of the release set filters on
this: the owner must resolve to a live publisher AND the document must sit
under that publisher's own feed branch. Without it, a document written
through the ordinary `DocumentApi.save` path — by whoever owns the
covering branch — would ride the ticker as a release, and the publishing
check would stop being the only way in.

## ⚠⚠ The write transport is an ownership bypass

`DocumentApi.save` gates on self-home / covering zone / slice-walk, which
admits the **parcel owner** — not the comms director. Making every comms
director a landowner is obviously wrong, so the press path writes through
**`DocumentApi.saveRelease`**, which stamps an owner that gate would never
admit (the `PersistableApi`-routes-capture-as-the-owning-principal
precedent).

Four things keep it narrow, and every one is load-bearing:

1. **Gated to one module** — `PressLogic`, and nothing else. ⚠ The gate
   lives on the **Api static**, not the logic method: every logic method's
   caller is its own Api face, so a policy there would name `DocumentApi`
   and narrow nothing (the `CompactApi.assignOffice` →
   `OfficeController` precedent).
2. **No caller-supplied owner.** It takes the publisher and derives one,
   so there is no parameter to lie in.
3. **Refuses a path outside that publisher's own feed branch.**
4. **Pins the `kind`**, so it cannot write anything else.

`mayPublishAs` sits in front of it. Authorship still records the *person*:
the transport appends a provenance row keyed on the path — the publisher
owns the document, a person wrote it.

## The surface — `PressApi` / `PressLogic`

The gated Api↔logic-singleton pair (`api/press.ts`,
`obj/api/PressLogic.ts` at `/obj/api/press`). The Api is a thin forwarding
shell ending in `SecurityApi.decorateApiClass`; the logic is `@internal
@Unshadowable extends ApiLogic` with every method gated
`FromModule('/api/press#PressApi')`, all real work in module-private
`*Impl` free functions.

- `publish(req)` → `Release`. The **author** derives from the execution
  context, never caller-supplied; the **publisher** is named by the caller
  and checked with `mayPublishAs` **before anything is minted or
  persisted**, so a refusal writes nothing at all. ⚠ There is no `realm`
  field on the request.
- `edit(releaseId, patch)` / `retract(releaseId)` — change-only-what-you-pass;
  retract is soft.
- `recent(limit?)` — the live ticker window.
- `archive(query)` — the paged history (`realm`/`kind` filters +
  `before`/`limit`), filtered in JS over the tree read.
- ⭐ `pressRoom(limit?)` → `PublicReleaseRow[]` — the **anonymous** read.
- `holdsAnyPublishingPosition(principal)` — the affordance-level boolean
  behind `requiresPublisher`. ⚠ It **selects nothing**, which is what
  keeps it from being the banned pick-a-publisher helper: that shape turns
  a refusal into a downgrade.
- `toRow(release)` → `ReleaseRow` — the pure projection shared by the
  frame fan-out and the session-establish window.
- `boot()` — a warm/activation seam for call-site symmetry; the board
  warms via its manifest `postRegister` and the fan-out is inline, so
  there is no event tap to install.

**`PressBoard`** (`obj/PressBoard.ts`, manifest-registered at
`/obj/PressBoard`) is a warm cache **over the tree** owning the window
semantics — `recentWindow()` = pins-first then `publishedAt`-desc,
excluding retracted and expired, the pin block capped at `press.maxPins`
and the whole window at `press.tickerWindow`. Soft-retracted releases stay
cached (reachable by id) and drop out of the window.

## Publishing — the `press` verb

`cmd/system/press.yaml` + `obj/command/system/PressController.ts`.
Subcommands `post` / `edit <id>` / `retract <id>`, with bare
`press <headline>` falling through to `post`.

- `--as <organization>` — **required and never defaulted.** Which masthead
  something goes out under is the author's statement; picking one for them
  would turn a refusal ("you hold no publishing position anywhere") into a
  silent downgrade ("...so here is the one you do hold").
- `--visibility` — narrows only.
- `--kind` — `changelog | decision | event | notice | repost`.
- `--source` — where a `repost`'s substance came from.
- `--pin`, `--expires` (shell-style `2h` / `7d` / `1h30m`).
- ⚠ **No `--realm`.** Derived.

### ⚠ `requiresPublisher`, and the break it fixes

The verb is gated by **`requiresPublisher`**, not `requiresAuthor`, and
the swap is a fix rather than a tidy-up.

`AccessApi.isAuthor` is membership of the `core` group — **which
`seedCoreGroup()` creates EMPTY**. `WIZARD_PLAYER_IDS` seeds `wizards`,
not `core`, and the only code path that adds anyone is a dev/test
provisioning helper. So on every fresh box *nobody* is an author, **the
founder included**, and the shipped gate refused the verb outright.

The affordance was never the barrier — `AuthorMixin` rides
`ShelledCharacter`, so every Avatar already carries the contribution. The
**validator** was.

## Reading it — three surfaces

1. **The live window — the `publication.press` frame.** On
   publish/edit/retract, `fanFeedImpl` fans a `ReleaseFeedFrame`
   (`upsert` / `remove`) to every online interactive over
   `PlayerApi.getAllAvatars()` (skip destroyed/linkdead, per-viewer
   `try/catch`). The fan is **inline** inside the mutators, and the
   payload is composed **once** — OOC-public, identical for every viewer,
   no per-viewer lensing. The current window is also delivered at
   session-establish: `Avatar.enter()` folds
   `PressApi.recent().map(PressApi.toRow)` into
   `ConnectionEstablishedPayload.releaseWindow`.
2. **The archive — `GET /api/press/archive`** (`requireAuth`).
   `?before=&limit=&realm=&kind=`, realm/kind validated (400 on a bad
   value), rows projected via `toRow`.
3. ⭐ **The press room — `GET /api/press/releases`** (**anonymous**).

### ⭐ The anonymous press room

Its own route rather than a flag on the archive: a route whose entire
contract is *"public, anonymous, no credentials"* is much harder to widen
by accident than a boolean on a route that also serves authenticated
readers. The handler reads no session, sets no cookie, and touches neither
`req.user` nor any cookie header.

⚠ **It rejects paging (`before` → 400)** rather than ignoring it. A
silently-ignored cursor is the shape of a future accidental widening. A
press room is not an archive; it serves the window.

⚠ **Two independent filters.** *Placement* — is this publisher named in
`press.frontPage`? — and *permission* — is this release publicly visible?
They are different questions and stay separately applied. A publisher can
be public and off the front page; a listed publisher can still have a
members-only release. Collapsing them makes placement imply permission.

⚠ **`PublicReleaseRow` is a standalone interface**, not a `Pick<>` of the
authenticated row and not an extension of it: structural sharing is how a
field added later leaks to the open internet without anyone noticing. Its
key set is frozen and asserted as frozen. **No `author`** — the
organization is the speaker, not the person who typed it — and **no
`expiresAt`**, which is operational metadata rather than press-room
content.

⚠ **No CORS change is needed and none should be made.** The app-wide
policy is single-origin with `credentials: true`, already covering
same-origin production and 5173→2010 dev; `requireAuthApi` is per-route,
never app-wide. A single-origin credentialed policy cannot have an origin
appended without reasoning about the credentialed routes it also governs.

## The client

- **`PressRoom`** (`components/PressRoom.tsx`) — the start screen's
  press room, beside the sign-in panel and **never awaited by it**.
  ⚠ `credentials: 'omit'`, spelled out because every other client fetch
  uses `'include'` and a copied idiom would send cookies to a route
  defined as reading none. One attempt, no retry, no polling;
  `AbortController` on unmount. **Three terminal states**: rows, an honest
  empty line, or render `null` — a visitor never sees an error string or a
  hanging spinner. MML renders with **no-op** `onCommandClick` /
  `onCommandPreview` (both are required props), so nothing reaches the
  command bus on a surface with no connection. Accepted residual: a
  clickable inside a release looks clickable and does nothing.
- **`NewsTickerPane`** — the post-login ticker on the client-owned third
  `rightPane` tab (`inspect | who | news`). The store `feed`/`feedOrder`
  slice mirrors the roster slice; `orderFeed` mirrors the server's display
  ordering so incrementally-`upsert`ed frames land deterministically,
  while the **authoritative semantics** (pin cap, expiry/retract
  filtering, window length) stay server-only.

Wire types in `@saxonberg/types`: `ReleaseRow`, `PublicReleaseRow`,
`ReleaseFeedFrame`, `ReleaseRealm`, `ReleaseKind`,
`ConnectionEstablishedPayload.releaseWindow`.

## Configuration

AppSettings ([app-settings.md](./app-settings.md), seeded from
`config/app-settings.yaml`, read via `AppApi`, no code defaults):
`press.tickerWindow`, `press.maxPins`, `press.headlineMaxLength`,
`press.bodyMaxLength`, and:

⚠ **`press.frontPage`** — which publisher organizations the anonymous
press room shows, as a **comma-separated** list of organization paths
(`AppApi.setting()` returns a `string`; there is no array type). An entry
that does not resolve is **skipped and logged**: a typo would otherwise
empty the front page with no signal, and "looking deliberate while empty"
is precisely this surface's failure mode.

**No publisher catalogue.** The list is short and enumerated, so each
entry is resolved by path; a warm catalogue would be machinery with no
query to serve.

## The seeded publishers

| path | realm | visibility | appointing authority | feed |
|---|---|---|---|---|
| `/compact/press` | `ooc` | `public` | `{kind: committee, parcel: /compact}` | `/compact/press/feed` |
| `/compact/executive` | `world` | `members` | `{kind: office, office: prime-minister}` | `/compact/executive/feed` |

Both ship **unfilled**, which is the design (decision 2). ⚠ The Compact's
**committee** authority is load-bearing: only `office` and `committee`
carry the Art. XI founder default, so an operator-axis authority there
would be one nobody could satisfy on a cold box.

⚠ The Office of the Prime Minister also ships with nothing to announce —
no legislature, no bills, no policy — so its releases are whatever is
written in character. **Cold start is content, not mechanism**: no NPC
outlet, no seeded demo release, no synthetic floor.

### ⚠ A press office is not a newsroom

Both seeded publishers are **institutional communications shops**, and
their position labels are the tell: *"speaking for the Compact"*,
*"speaking for the Prime Minister"*. A newsroom role is never labelled
with the name of its subject.

So the roster here is `communications-director` (and, at the executive,
a `press-secretary` reporting to it) — **not** an editor in chief. That
is doctrine, not taste: **the state aggregates, never reports.** The
state may cite; only a publisher may claim. It is the rule that struck
the herald seat from the gazette slate's Wave 1. Putting an editor in
chief on a government press office would be the state standing up a
newsroom and calling its own output journalism, which is precisely the
failure the rule exists to prevent.

The newsroom roster belongs to a **different organization that does not
exist yet** — the vocation in
press-slate, the one that makes
transparency actual rather than declared. Its shape when it lands:

```
publisher → editor-in-chief → editor → reporter
```

...with `publishingPositions` most likely stopping short of `reporter`
(a reporter *files*; an editor *publishes*), and `reportsTo` finally
carrying a chain deeper than the two links the executive uses today.

⚠ **A vocabulary collision to expect there.** This subsystem uses
"publisher" to mean *a thing that publishes* — `PublisherMixin`,
`publishingPositions`, `getFeedPath`. In journalism **Publisher is a
job**: the business side, the person who hires the editor in chief. So a
newspaper will be a `Publisher` (mixin) whose positions include a
`publisher` (role). Awkward, and deliberately left alone — the two live
in different namespaces, and the mixin's meaning is already load-bearing
across `/compact/press`, the executive, and any future organization that
publishes without being a news outlet at all.

## Non-goals

- **No threading / replies / reactions / voting** — two-way deliberation
  is the forums' job ([forums.md](./forums.md)).
- **No active delivery** — no badges, pokes, or `lastSeenAt`.
- **No edit trail** — edits are in place.
- **Not wired to the provenance ledger** as *authorship of a release*:
  `author` is a stored identity string. (The write transport does append
  a provenance row keyed on the document path.)
- **Not MQL-queryable** — a platform feed, not in-world Stuff.
- **No subscription, push, byline, credibility track record or paywall** —
  that is the press build (press-slate).
- **No permalinks, `og:` tags, RSS or email**, and no panterasbox.com
  consumer — the surface is the start screen.

## The rename inventory (from `bulletin`)

| from | to |
|---|---|
| `lib/bulletin/` | `lib/press/` |
| `lib/bulletin/Bulletin.ts` | **deleted** — a release is a `StoredDocument` |
| `obj/BulletinBoard.ts` @ `/obj/BulletinBoard` | `obj/PressBoard.ts` @ `/obj/PressBoard` |
| `api/bulletin.ts` / `BulletinApi` | `api/press.ts` / `PressApi` |
| `obj/api/BulletinLogic.ts` @ `/obj/api/bulletin` | `obj/api/PressLogic.ts` @ `/obj/api/press` |
| `backend/BulletinRoutes.ts` | `backend/PressRoutes.ts` |
| `cmd/system/bulletin.yaml` + `BulletinController` | `cmd/system/press.yaml` + `PressController` |
| `BulletinRow` / `BulletinFeedFrame` | `ReleaseRow` / `ReleaseFeedFrame` |
| `ConnectionEstablishedPayload.bulletinWindow` | `.releaseWindow` |
| Topic `publication.press` | `publication.press` |
| AppSettings `bulletin.*` | `press.*` |
| the `bulletins` collection | **retired** — the tree |

⚠ Two carry **stored state**, and the choice was deliberate rather than
incidental: the seeded **Topic** the client subscribes to by name, and the
**AppSettings keys** (renaming leaves orphan `app_settings` rows — unread,
harmless, untidy). `NewsTickerPane` keeps its name: it is a ticker, not a
bulletin.

## Cross-references

[employment.md](./employment.md) (the organization chart a publisher wears
its hat over; `mayPublishAs`),
[document-store.md](./document-store.md) (the `release` kind + the named
write transport + the four-namespace taxonomy),
[parcel.md](./parcel.md) (the `/compact` title),
[governance.md](./governance.md) (offices, the founder default),
[access.md](./access.md) (`AccessApi.isAuthor` — and why it is not the
gate here), [social-graph.md](./social-graph.md) (the roster-frame fan
precedent), [topics.md](./topics.md) (the session-establish snapshot
seam), [forums.md](./forums.md) (the `{text, fields}` body side-channel),
[help.md](./help.md) (the read-only REST data-API precedent),
[app-settings.md](./app-settings.md),
[client-shell.md](./client-shell.md) (the start screen),
[message-rendering.md](./message-rendering.md).
