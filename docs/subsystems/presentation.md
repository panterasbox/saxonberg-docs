# Presentation — what a thing is called, and which form of it a sentence needs

> **A thing presents a NOUN PHRASE, not a string with an article typed
> into the front of it.** And **which form** of that identity a sentence
> gets is the sentence's choice, resolved late, beside whoever is reading
> it.

Two axes, both late-bound, both answering the same question from
different sides: *who is reading* and *what does this sentence need*.

---

## 1 · The noun phrase

`lib/description/NounPhrase.ts` — a value object: **a stem, a register,
a count**.

```ts
GrammarApi.phrase('collie', 'definite')   // the collie
  .withCount(2)                           // 2 collies
  .possessive()                           // 2 collies'
```

⚠ **Built through the Api, never `new` and never a static on the value
class.** Construction is an Api concern — the same rule that sends
`new SomeStuff()` through `StuffApi.create` — and the reason is
mechanical: the author-surface projection admits public **Api statics**
and public **instance methods**, so a static on a `lib/` class is
*callable by anyone and visible to nobody*. ⭐ 24 other value classes in
this tree still break that; see
value-object-statics-slate.

`Stuff.getPresentation()` is `presentationPhrase().render()`. Its
signature never changed, so all ~265 callers, the wire's `displayName`,
`MessageApi.refOf`, MQL scalars and the logs are untouched — a string is
the right type for a sink that only prints.

### The register

| register | reads as | who |
|---|---|---|
| `proper` | `Odile` | somebody, with a name |
| `definite` | `the collier` | somebody, without one |
| `indefinite` | `a sentry` | a role, one of many |

⭐⭐ **This is not a new taxonomy.** The content encoded it by hand **611
times**: every authored `shortDescription` began with an article
somebody typed. And the two identity rungs are *defined* by it — the
*somebody* rung's own documentation leads with *"**The** collier"*, the
*role* rung's with *"**A** sentry."*

⚠ **The field lives on `VisibleMixin`, not on the rungs**, and that is
deliberate: **480 of the 611 articled rows are things and locations,
which have no identity rung at all.** So the field sits beside the
description it qualifies, and the rungs enforce *agreement* —
`lint:identity` rule 2 reads `register` to check that an `Extra` is *a*
sentry and a nameless `Cast` is *the* collier.

The default is `indefinite`, because 476 of the 611 rows are.

### ⚠ A stem never carries an article

`heavy iron door`, not *"a heavy iron door"*. A stem that smuggles one
in renders *"a a heavy iron door"*. `lint:presentation` clause (a) gates
it at a hard zero, and it walks pack `src/*.yaml` as well as template
rows — `dorm-themes.yaml` holds 28 authored descriptions that reach
`setShortDescription` and is not a template row.

**Every RUNTIME writer of the field writes a stem too** — the corpse
overlay, the coat-check ticket, three crafting lumps, four key
descriptions, a cast lump, a drafted animal. There were nine, and each
would have doubled the article.

### What it made possible

Three sentences that could not be written before without editing a
string: *the collie*, *two collies*, *the collie's paw*. And one that
was quietly wrong: `GrammarApi.article` read the article off the
**rendered** string, so it answered `'an'` for *"a heavy door"* and
`'a'` for every proper name. It asks the phrase now.

---

## 2 · The form

`PresentationForm` — six values, resolved at render time beside the
viewer.

| form | shows | wanted by |
|---|---|---|
| `bare` | the name alone | chat, when anonymity is off |
| `handle` | article + the short handle | chat, when anonymity is on |
| `concise` | the ordinary identity | act lines, emotes — most prose |
| `presence` | + what they are doing | the room survey |
| `distinguishing` | + what they are wearing | targeting, disambiguation |
<!-- ⚠ The disambiguation PROMPT does not use this form yet: the fishing
     build's live drive typed `look cane` in front of a shelf holding two
     cane rods (`par: 2`) and was offered two buttons both labelled *a
     cane rod* — a choice a player cannot make. Either render the choices
     in the `distinguishing` form, or collapse identical candidates into
     one. -->
| `formal` | the full name, honorific and suffix | profiles, documents |

An emitter asks for one: `Mml.actor(speaker, { form: 'presence' })`. The
seam resolves `stuff.describeFor(viewer, form)` at
`Mml.toString(viewer)`, per recipient.

### ⭐⭐ Why this is the interesting half

Before it, **the rich form and the multi-recipient broadcast were
mutually exclusive**. You could have status and distinguishing features
(one viewer, resolved eagerly) or per-recipient naming (concise only),
never both — so the room survey could say what people were doing and an
emote could not. That looked like a design decision and was an
implementation consequence wearing its costume.

The proof it was doing damage: the two richer faces had **one caller
each**, both on the same surface.

### ⚠⚠ `bare` and `handle` consult no perception gate

This is the disguise/anonymity split, in two lines of code.

- **A disguise** is a *visual fact*, defeated by perception. It works
  because somebody is **looking at you**.
- **Anonymity** is a *declared stance about a message*, independent of
  what anyone can see.

A channel is not looking at you — it is reading what you typed. So **a
hooded man on a named channel is himself**, and an anonymous poster is a
handle no matter how plainly he is standing there. The two were one
mechanism, which is why the question had no answer rather than a wrong
one.

⭐ A `handle` names nobody, so its wire tag carries **no `stuff-id`**.
The tag kind is the object's own truth (`player` for a player): that a
weaver is a player is not the secret; *which* player is.

### One ladder, two views

`Stuff.presentationPhrase(view)`:

1. **the disguise** (`appearsAs`, always indefinite — being one of many
   is what a disguise is for)
2. **a proper name**, if the object can hold one — `proper`
3. **the authored description**, in its register
4. **the species common name** — `indefinite`
5. `something` / `someone`

`'stranger'` skips rungs 1 and 2. ⚠ That ladder used to exist **twice** —
here, and in `RecognitionLogic.strangerStem` as the same rungs minus two,
kept in step by hand. The first change to touch one broke the other.

---

## 3 · The handle

*What kind of thing this is, in one or two words, with an article.*
`Stuff.handlePhrase()`:

1. the handle **its author wrote** (`getAuthoredPrimaryKeyword()`)
2. else **the role they hold** (`Employed.getPositionNoun()`)
3. else **their species**
4. else the description stem

⚠⚠ **Rung 1 reads the *authored* slot, and that is an anonymity
guarantee rather than a nicety.** `getPrimaryKeyword()` always answers
something, because targeting needs it to — and its derived pool folds in
`tokenizeName(getName())` for any `Named` host.

A player body is `Named` (embody writes it), `Perceptible` (from
`Creature`), and carries **no authored keyword**, because nobody types
one for a player. So the derived answer for a player called Odile is
**`odile`**, and a handle chain reading it would sign her anonymous post
*"an odile"* — the single fact the whole setting exists to withhold.
`getAuthoredPrimaryKeyword()` returns `undefined` there, the chain falls
to her species, and she posts as *"a human"*.

⭐ Two different questions, two right answers: *what may I type to refer
to this?* derives freely; *what does this call itself?* is only ever what
somebody wrote.

⭐ **Rung 2 follows the job**, and that is a bug fix rather than a
convenience: a handle retyped onto the NPC goes stale, which is why a
dismissed weaver still reads *"a weaver"*. `Position.noun` is a second,
optional word beside `label` because **every shipped label is a gerund**
(*"tending bar"*, *"on the road"*) and there is no honest
gerund-to-noun transform.

### ⚠⚠ The flavour never derives

The world may derive **what kind of thing** something is. It may never
derive **what it is like**. *"A dyer whose hands are blue to the wrist
and will not come clean"* is not derivable from any model, and a system
that could generate it would give every world the same portraits.

---

## 4 · Chat anonymity

`Channel.anonymity` — `permitted` (default) or `forbidden`.

| channel | post | shows |
|---|---|---|
| `forbidden` | plain | the **name**, for everyone, hood or no hood |
| `forbidden` | `--anon` | the **name** too — the flag is declined, and the poster is told |
| `permitted` | plain | today's concise identity, byte-identical |
| `permitted` | `--anon` | a short handle; no `stuff-id`, no speaker ref |

Set with `chat anonymity <name> permit|forbid`, owner-only. ⭐ The one
thing in this build a player can see change — and a real values question
about a community, now expressible.

⭐⭐ **`--anon` is a request the channel may decline, not a precondition
that fails.** You always get to speak; the setting decides whether your
name shows. ⚠ And the decline is said out loud — somebody typed `--anon`
for a reason, so a line telling them it went out named is the difference
between a rule they can see and a surprise they find afterwards.

⚠ An anonymous post **omits** the speaker ref rather than blanking it: a
`StuffRef` carries a `stuffId`, and a client holding one can ask the
world about it. The rendered line is not the only place a name leaks.

⚠ `anonymity` is in `ChannelCatalogue.RESERVED_NAMES`. The bare post is
a **fallthrough**, so an unreserved subcommand silently steals a channel
name somebody could already hold.

---

## The acceptance bar, and how it was held

> **A player cannot tell this happened.**

Three independent instruments, because this was the widest-blast-radius
change in the repo's recent history:

1. **A content golden** — `check-presentation --snapshot` / `--verify`,
   one extractor over 635 content rows spanning both sides of the
   codemod: it read a pre-sweep row (`a heavy door`) and a post-sweep row
   (`heavy door` + `register: indefinite`) and had to answer the same
   string. A golden captured **before** the sweep therefore proves the
   sweep. ⚠ **Retired at the sweep, deliberately** — a golden is only a
   proof while nobody may re-record it, and the next person to change a
   row would have to. The gate's permanent clauses stayed; the two modes
   and the fixture went.
2. **Unit tests** — the render matrix, the ladder, the form matrix.
3. **The live transcript diff** — `look` in five rooms, before and after.

⭐⭐ **And the third one earned its place.** The golden stayed green
through a live regression: `Visible.getLong()` fell back to the **raw**
description, so `look <thing>` printed *"brass altimeter"* into its body
where a player had always read *"a brass altimeter"*. The golden reads
YAML and never renders a body. There **was** a unit test on that
fallback — and it asserted a *field round-trip* (set `'A rusty sword'`,
expect `'A rusty sword'`), which is true of the raw field and says
nothing about the prose.

> ⭐ **A test that feeds a value in and asserts the same value out cannot
> see a rendering change.**

⚠ And a transcript diff is a **noisy** instrument: of its four
differences, one was the bug, one was NPC brain ticks, one was the
light/visibility gate moving with the game clock, and one was a fix
landing mid-run. Re-run it twice on the same code before blaming a diff
on your change, and normalize `stuff-id` — they are re-minted on every
pack install.

---

## The one deliberate departure

`Mml.list` was **eager**: every branch called `toString()` with no
viewer, so late binding died the moment a reference went through a list.
`sense`, `search` results, on-surface and in-container lists named a
person you had never met. The room's own roll-call escaped only because
it resolved eagerly per viewer before listing.

Making it lazy closes that by construction — no call site was edited —
and it *is* visible for a disguised or unrecognized person in those
lists. It is the behaviour `belief.md § The prose path` documents as the
contract; the alternative is a second lazy list for occupants only,
which is a guard.

---

## Where the pieces live

| piece | file |
|---|---|
| the value object + both vocabularies | `lib/description/NounPhrase.ts` |
| **building one** (`phrase` · `properPhrase` · `articleFor`) | `api/grammar.ts` |
| `register` | `lib/description/Visible.ts` |
| `presentationPhrase` · `handlePhrase` · `describeFor(viewer, form)` | `lib/stuff/Stuff.ts` |
| form resolution on the wire | `api/mml.ts` |
| the recognition face | `platform/idea/api/RecognitionLogic.ts` |
| the occupant block | `platform/idea/api/SocialLogic.ts` |
| `noun` · `getPositionNoun` | `lib/employment/{Position,Employed}.ts` |
| `anonymity` | `lib/social/Channel.ts`, `platform/idea/ChannelCatalogue.ts` |
| the gate | `scripts/check-presentation.ts` |

**See also:** [messaging](./messaging.md) (the render seam) ·
[belief](./belief.md) (recognition, disguise) ·
[identity](./identity.md) (the rungs) · [chat](./chat.md) ·
naming slate (minimal-distinguishing
rendering — claimed in a docstring for years, **never built**).
