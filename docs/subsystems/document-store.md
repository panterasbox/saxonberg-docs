# Document store (the owned-JSON tree)

The document store is the **third path-addressed tree** — the slate's
**source / template / document** triad. It is the generic, owner-claimed
tree of arbitrary JSON the runtime hands a user to fill: claim a branch
and put any JSON there, tagged with a `kind` that says what kind of
object lives at that path.

It generalizes the original scripting player-script store. Scripts are now
just **one kind** of stored document; dorm-room customization will be
another (the first non-script consumer — see the Warren/lounge work).

## What a record is

`StoredDocument` (`lib/document/StoredDocument.ts`, collection `documents`):

| field   | meaning |
|---|---|
| `path`  | canonical key; owner/scope encoded in the path — see the namespace taxonomy below |
| `owner` | the owner's durable `templatePath` (set from context, never the caller) |
| `kind`  | **what kind of object lives here** — `'script'`, later `'dorm'`, … |
| `data`  | the arbitrary JSON payload |

The store is **kind-agnostic**: it persists and serves `{path, owner,
kind, data}` and never inspects `data`. Each `kind`'s consumer owns the
*meaning* of `data` and any **go-live** behavior (re-parse, re-hydrate).

`StoredDocument extends Document` only for persistence — distinct from the
persistence `Document` base class it shares a word with: a `Document` is
*how* a row is saved; a `StoredDocument` is the *owned-JSON-in-the-tree*
concept.

## ⭐ The four namespace kinds

The path prefix says what kind of thing a branch **is**, and there are
four — not three. The fourth was surfaced by needing somewhere for the
Compact to publish, and the taxonomy did not have it:

| namespace | what it is |
|---|---|
| `/home/<self>` | **workspace** — personal |
| `/studio/<group>` | **workspace** — the multiseat `/home` |
| `/domain/<locality>` | **content** — a *place*, with rooms and NPCs, locally administered |
| `/compact` | **an institution's own branch** — the Compact's; it keeps publications there |
| `/corpo/<key>` | **an institution's own branch** — a company's |

> **An institution's branch is what it owns that is not a place.**

⚠ **The fourth row was first written as "publications", and that was
describing the Compact's *contents* rather than the *kind*.** Giving each
corpo a branch of its own showed what the kind actually is: a body that
is not a place still needs ground it holds title over — because that title
is what makes it a **committee**, and a committee is how it appoints
([corpo.md](./corpo.md), [employment.md](./employment.md)). The Compact
keeps publications on its branch; a corpo keeps a chart on its. Neither
makes rooms.

Two wrong turns are recorded here because the next person will reach for
them too:

- ⚠ **`/studio` is not the precedent for a publishing namespace.** It is a
  *workspace* namespace, the multiseat `/home`. Its parcel **row shape** is
  the right model — a core-held path-branch title deliberately omitting
  `landUse`/`areaM2`, so the inheritance walk answers `wild` and admits
  nothing — but its *concept* is not.
- ⚠ **`/compact` is not a federal enclave.** The diegetic premises of
  Compact institutions are Saxonberg's rooms, **administered locally** (DC
  has its own police under a seated city government). `/saxonberg` will be
  *content*. `/compact` is not where the buildings go; it is where the
  documents go.

See [parcel.md](./parcel.md) for the `/compact` title.

## The Api (`api/document.ts` → `obj/api/DocumentLogic.ts`)

A thin gated forwarding shell over a hot-reloadable logic singleton at
`/obj/api/document`, the `ScriptLogic`/`CraftingLogic` precedent:

- `DocumentApi.read(path)` → the `StoredDocument` at `path`, or null.
- `DocumentApi.list(prefix)` → every doc at/under `prefix` (the CMS tree's
  input).
- `DocumentApi.list(prefix)` / `DocumentApi.listOfKind(kind)` — the second
  is the release window's rebuild input.
- `DocumentApi.save(path, kind, data)` → find-or-create. **Access-gated**,
  **owner-stamped from context**, **provenance-recorded** (below).
- ⚠⚠ `DocumentApi.saveRelease(publisher, path, data)` — **the named write
  transport**, and an ownership bypass by construction. See below.

The acting **owner is always derived from `ExecutionContextApi`** (the
in-world command-frame giver, or a transport's `tagActingAuthor` stamp),
**never a parameter** (memory: gated-api-actor-from-context).

### The gate — self-home ownership

`DocumentApi.save` reuses the whole ownership stack, with one base case on
top:

1. **An owner owns their own `/home/<self>/` branch** — keyed on the
   durable-path basename, so a player owns exactly the subtree the runtime
   banks under their name (recorded recipe-scripts today, a dorm's
   customization tomorrow). No broader grant needed. As of property phase
   0a this rule is **shared, not forked**: `isOwnHomePath` consumes
   `ParcelApi.selfHomeOwnerOf` (the single implementation, also rung 2 of
   the parcel `ownerOf` chain — see [parcel.md](./parcel.md)).
2. else the covering spatial zone gates via `AccessApi.canMutateZone`;
3. else the slice-walk `AccessApi.can(write)`.

### Provenance

Every save appends an `AuthoringEvent` keyed on the path —
`DocumentLogic` (`/obj/api/document`) is a named authoring transport in
the `ProvenanceApi.recordAuthoring` gate (alongside the template
chokepoint). Authorship is *derived*, not a mutable stamp.

## ⚠⚠ The release write transport — a named ownership bypass

`kind: 'release'` documents ([press.md](./press.md)) are owned by the
**publisher organization**, not by the person who wrote them. The gate
above cannot express that: it admits the **parcel owner**, and making
every comms director a landowner is obviously wrong.

So the press path writes through `DocumentApi.saveRelease`, which stamps
an owner the ordinary `save` gate would never admit — the same shape as
`PersistableApi` routing capture as the owning principal. **If it is
loose, the whole document-store ownership story is broken, not just for
releases.** Four things keep it narrow, and each is load-bearing:

1. **Gated to one calling module** (`PressLogic`). ⚠ The gate lives on the
   **Api static**, not the logic method: every logic method's caller is
   its own Api face, so a policy there would name `DocumentApi` and narrow
   nothing (the `CompactApi.assignOffice` → `OfficeController` precedent).
2. **It takes no caller-supplied owner** — it takes the publisher and
   derives one, so there is no parameter to lie in.
3. **It refuses a path outside that publisher's own feed branch.**
4. **It pins the `kind`**, so it cannot write anything else.

⚠ And because the tree is a **shared store**, the complement matters as
much: `kind: 'release'` is a tag anyone who can write a document can
apply, so **every read of the release set re-verifies what the transport
guarantees** (`Release.publishedByOwner` — the owner must resolve to a
live publisher, at a path under that publisher's feed). Without it, a
document written through the ordinary `save` path by whoever owns the
covering branch would ride the ticker as a release.

## Scripts as a `kind`

`ScriptLogic` keeps the script **semantics** — parse source → AST, the
per-path AST cache, the script-specific go-live — and delegates **storage**
to `DocumentApi`:

- `ScriptApi.saveScript(path, source)` → `DocumentApi.save(path, 'script',
  { source })`, then invalidate the AST cache (the script go-live).
- resolve-by-path → `DocumentApi.read(path)`, take `data.source` when
  `kind === 'script'`, parse + cache.

The generic store deliberately does **not** keep an AST cache (an AST is
script-specific) and runs **no** kind-specific go-live — that all lives
with the kind's consumer.

## CMS — the third tree

The CMS exposes the store as the **`'document'`** backend (`CmsBackend =
'content' | 'source' | 'document'`), the third root in the explorer
alongside content (templates) and source (engine TS). The record's `kind`
drives the editor treatment:

- `kind: 'release'` → a press release ([press.md](./press.md)), under its
  publisher's declared feed branch, owned by the publisher.
- `kind: 'script'` → the source text (`data.source`) as a plain-text code
  leaf; a write funnels through `ScriptApi.saveScript` (the script
  chokepoint: gate + provenance + AST go-live). Scripts are the one
  runtime-**creatable** kind.
- any other kind → `data` pretty-printed as JSON; a write parses the JSON
  and persists via `DocumentApi.save` under the doc's existing kind (no
  live re-hydration consumer yet — that lands with dorm).

Author-tier read gate (like content). Wired end-to-end: the REST
`parseBackend` admits `'document'`, and the client explorer lists the
`documents` root.

## Deferred

The fuller per-`/home/` access model (sharing, grants beyond the
self-owner), non-script document **creation** via the CMS, and the first
non-script consumer (dorm-room customization data, hydrated onto a Warren
constituent — base-template + customization-document) all land later. The
substrate is built to accommodate them; only scripts ride it today.
