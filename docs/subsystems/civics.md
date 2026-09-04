# Civics — diegetic government

The **Government** substrate: governments *inside the fiction* as
plural, authored content. A `Government` is a pure-data reference-
identity (the `Corpo` recipe) authored by a committee; **jurisdiction
is declared on the `Locality`** and derived as a chain over the address
coverage walk; **residency derives from a character's domicile**;
**seats are employment positions**. Seeded by the diegetic-government
design capture (docs/staging/diegetic-government.md
— the two-layer premise, the six enforcement powers, the jargon
standard); the Terminus government stack is
locked in terminus-city.md §8. (The
build's requirements + plan were retired at the pre-merge sweep, per
the workflow.)

**The premise (doctrine, not deferral):** the Compact — the real
meta-institution — governs the platform and is singular; diegetic
governments are content, mintable per locality at every tier (realm
included — realms are plural), owned via the property bridge, and
enforceable only through the six landowner powers. **No tier of the
fiction's hierarchy is the Compact's face.** This substrate builds
reads and identity — **no legal machinery** (no statute engine, no
trials, no arrest mechanics): enforcement is content built on existing
substrates, and diegetic justice moves real-substrate value only
through the real chokepoints.

Source: `obj/Government.ts`, `obj/GovernmentCatalogue.ts`,
`platform/idea/api/GovernmentLogic.ts`, `api/government.ts`,
`cmd/civics/government.yaml` + `platform/idea/cmd/civics/GovernmentController.ts`.
The meta-side committee concept lives on **`CompactApi`**
(`api/compact.ts`) — see [access.md](./access.md) § The committee.

## The Government Idea + catalogue

`Government` (`obj/Government.ts`, templates under
`<root>/idea/Government/<key>`) is a pure-data leaf `Idea` read from
`template.data`, never cloned live. Fields — every non-identity field a
**durable-string reference into an existing substrate**, never a live
ref:

- `key` (the durable join — a `Locality` declares its government by
  key), `displayName`, `description`;
- `charter` — a document-store path (pointer only in v1: nothing reads
  it yet, no StoredDocument is seeded — deferred);
- `treasury` — a bank-account key (the city points at the shipped
  municipal budget `/world/terminus/budget`);
- `departments` — **organization** templatePaths (an organization's path
  IS its durable key in the employment substrate). ⚠ They were Business
  paths, and that was the conflation the organizations build fixed: a
  registry keeps records and does not trade, and was a Business purely
  because that is where positions lived. Terminus's Registry is still a
  Business and resolves unchanged; new departments name
  `/platform/idea/Organization`.
- `seats` — `{ key, label, department, positionKey }` references: **a
  seat is a position on an organization's chart**, never a second Office
  apparatus (the real polity's code-authored Office substrate is
  deliberately not reused — see the two-staffs contrast below). ⭐ The
  seat/staff line is exactly whether a constitutional document *points
  at* the position: a seat is pointed at, personal staff is not — see
  [employment.md](./employment.md).

`GovernmentCatalogue` (`/platform/idea/GovernmentCatalogue`, manifest-warmed) is
the corpo-recipe data-cache: one boot query, sync reads, defensive
copies, cold-state-empty, singleton destruct/evict refusals. Like the
corpo catalogue it has **no auto-invalidate** on template churn beyond
HMR re-clone — a CMS edit to a Government template needs
`dest /platform/idea/GovernmentCatalogue`.

## Jurisdiction on the Locality (consent-by-construction)

A `Locality` carries the sparse tier field **`_governmentKey`** (the
third realized tier field — weather pin / climate lean siblings;
`null` is the common case, the setter trims empty to `null`). The
government carries **no claims list**: declaring jurisdiction is
authoring the Locality, which is already landowner-gated content
authorship — the property-bridge legitimacy rule ("a claim is valid
iff the landowner consents") needs no separate mechanism. Federation
is pointing at an existing key; secession is repointing.

The chain derives per read: resolve the address, walk
`AddressApi.coverageChainOf` (the sync chain sibling of
`coveringLocalityOf`, added by this build), collect declared keys
most-local first, drop `null`s and keys unknown to the catalogue,
dedupe preserving the most-local position, resolve to descriptors.

## The reads (`GovernmentApi`)

| read | sync? | semantics |
|---|---|---|
| `governmentAt(address)` / `governmentChainAt(address)` | sync | trie + in-memory catalogue (the `coveringLocalityOf` fast-path doctrine) |
| `subjectTo(scope)` | async | the chain over where the scope IS — what binds a visitor (the full resolve walk's zone step awaits) |
| `residentOf(character)` | sync | the chain over the **domicile** — membership standing (the future franchise/petition/services hook) |
| `domicileAddressOf(character)` | sync | the residence seam's face |
| `holdsSeat(character, govKey, seatKey)` | async | the authority predicate (the `requiresGovernor` analogue as data) — live `Employment` records first (active statuses win; an explicit quit/fired record **suppresses** the roster path — an exit is never resurrected), then the authored roster (the live organization if standing, else the template row — what makes a lazily-stood-up department's seat provable) |
| `seatsOf(govKey)` | async | each seat resolved to its roster holder |

**Never-throws:** every read returns `[]`/`null`/`false` off-grid, for
an unclaimed place, an unknown key, or a dead store — no government is
a normal state of the world (the Compact's own future capital district
is *deliberately* extraterritorial this way).

## Residency + the domicile seam

Modeled on real municipal residency-by-domicile: nothing confers
residency, no rows are stamped — `residentOf` derives the chain from
the character's home address. `Character` carries the sparse
`_domicileAddress` (Pattern-A address string, on the Avatar snapshot
via `fieldMeta`'s persistent entries). **Persists-until-replaced is structural**: the
setter ignores clearing writes; a new home overwrites; losing a
dwelling leaves it standing (homelessness is no dwelling, not no civic
identity — asserted in the residence suite: unprovision leaves the
domicile).

Writers are **residence content**: `ProvisionController` stamps the
tenant with `DormRoom.ADDRESS` at lease-grant (best-effort — a stamp
failure never voids the lease). There is no admit-time self-heal for
pre-build tenants (the Warren has no clean view of the mover) — the
known migration gap; a pre-build tenant re-provisioning gets stamped.
NPCs declare `_domicileAddress` in their seeds (Odile, the registrar,
is the exemplar). Multi-residence primary-home designation is deferred
until a second home type exists.

## The flagship — the three-tier Terminus stack

The state-capital model (terminus-city.md §8): three governments, one
re-rooted address chain, jurisdictions never fused:

| tier | Locality (claims) | Government key | built |
|---|---|---|---|
| realm | `terminus` (the root — still the departures-board label) | `terminus-realm` | thin: identity + jurisdiction |
| city | `terminus/city` | `terminus-city` | **wired**: Registry department, Magistrate seat, treasury = the municipal budget |
| campus | `terminus/city/campus` | `eternal-university` | thin (the town-gown layer) |

A dorm room (`terminus/city/campus/duncan-hall`) resolves
**[university, city, realm]**; the terminal (`terminus/city/…`)
resolves **[city, realm]** — sparse inheritance on real content. The
**charter-text-vs-template-data split**: the city's full commission
form (Administrator, Treasurer, Watch, Works, Almonry) is charter
*prose* in terminus-city.md §8; the template carries only what is
built — the Registry (a municipal-shape Business at
`/world/terminus/registry/business`; Odile double-hats registrar +
Magistrate on a shift-less roster, making `holdsSeat` live-provable)
and its rented annex off the arrival gate's east frontage. The
registry parcel is `terminus`-group-held — a `requires.title` claim of
the transitional `world-seed` pack (`/world/terminus/registry`,
`landUse: civic`), since content-packs wave 3 retired `config/parcels.yaml`
and the seeders ([content-packs.md](./content-packs.md)).
Sibling address roots (`counting-houses`, `university-avenue`,
`lounge`, `last-counted-mile`) remain **under no government** —
deliberate closed scope (re-rooting them is city content), not a bug.

## The verb

`government` / `gov` (the **`civics`** category — the fiction's
governments; deliberately not `governance`, the Compact's category),
universal on Persona, public, read-only: bare/`list` renders the
`subjectTo` chain most-local first (description, departments, seats
with holders, charter, a `mudcmd:` residency link); `residency`
renders the domicile line + `residentOf` chain. Empty chain and
undomiciled paths get honest prose + structured notes. NPCs can
dispatch it (no Interactive assumptions).

## The two staffs (never merged)

- **Meta staff** — the committee: parcel owners + code-trusted wizards,
  on the property/code-trust axes (`CompactApi.committeeOf` — the
  title-holding group IS the committee; see access.md).
- **Diegetic staff** — seat-holders: players or NPCs, on employment
  positions.

Authority moves across the boundary by **swapping the validator on a
verb** (the planned Circulation-Reserve migration: `requiresGovernor` →
a `holdsSeat` check on a future city Treasurer), never by merging the
office systems.

## Deferred / never

**Never (doctrine):** legal machinery — statute engine, trials, arrest,
sentencing; a Government mixin on Stuff; claims lists on governments.

**Deferred:** the shelter + gaol + Watch content (city content, with
arrest behavior designed against the consent substrate); marriage /
registry record kinds (Documents); seat elections/terms; taxes and
franchise consumption (the residency reads are the hook); a generic
`holdsSeat` validator (extracted at the third consumer, with the
deferred generic office validator); the charter as a readable
StoredDocument; multi-residence primary-home designation; re-rooting
the sibling address roots; the University's institutional detail
(Chancellor, colleges); the realm's ministries; the Compact's capital
district (staging).

## Cross-references

- docs/staging/diegetic-government.md
  — premise, six powers, jargon standard, capital-district future
- docs/staging/terminus-city.md §8 — the
  government stack + the city's charter-text structure
- [address.md](./address.md) — Locality, the coverage walk, tier fields
- [corpo.md](./corpo.md) — the data-Idea/catalogue recipe
- [employment.md](./employment.md) — organization/Position (departments,
  seats)
- [residence.md](./residence.md) — the dorm home the domicile stamp
  rides
- [governance.md](./governance.md) — the Compact's Office substrate
  (the deliberate contrast)
- [access.md](./access.md) — the committee (meta) + the property bridge
- [banking.md](./banking.md) — the treasury chokepoints

## Government ground — the `arm` property gate

The one verb that asks "is this public ground?" is `arm`
(`ArmController`, the trap-placement gate). With no state default in
the parcel layer (content-packs wave 3), *public* is no longer "held by
the state group" — it is **government ground with no private title**:
the giver may rig where they hold (`AccessApi.can` write), or on ground
under a government's jurisdiction (`GovernmentApi.governmentAt(address)`
non-null) that nobody privately holds (`ParcelApi.ownerOf` null).
Refused on another owner's property and on ungoverned ground.
