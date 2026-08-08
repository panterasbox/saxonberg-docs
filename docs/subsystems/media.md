# Media

The engine's model of **non-text renderable content** — the images (and,
later, audio/video/embeds) the client surfaces beyond the MML text stream.
Text rides the messaging/rendering subsystem; *this* covers everything
non-text. The organizing principle: the engine never *renders* this
content — it carries a **reference** to it (and, for content we generate
ourselves, a provenance record); the client does the rendering.

Two flavors, one umbrella:

- **Hosted assets we produce** — generated illustrations, hosted in S3,
  referenced by a bucket-relative key, and tracked (`MediaAsset`).
- **External sources we embed** — Twitch, third-party video, lesson
  media. Carried by provider + id/URL, no provenance. *Not yet built*;
  the subsystem is named for the umbrella so they have a home.

## The reference: `Visible.illustration`

An illustration is a *visual description*, so it lives on
`VisibleMixin` (`lib/description/Visible.ts`) next to
`shortDescription` / `longDescription` — a nullable, key-valued property:

```ts
getIllustration(): string | null
setIllustration(value: string | null): void   // setter invariant: "" → null, strips leading slash
```

The value is a **bucket-relative key** (`location/duncan-hall-lobby.png`),
never a URL. The server stays bucket/CDN-agnostic; the bucket can move
without touching content. `Species` composes `Visible`, so a *kind*
carries its portrait the same way an instance does (the char-gen race
picker reads `species.getIllustration()`).

## The wire: it rides the existing projection

`illustration` is one entry in `DETAIL_FIELDS`
(`api/mql-subscription.ts`); `VisibleMixin` contributes the
`SubscribableFieldDescriptor` exactly as the descriptions do, and it
lands on `StuffDetailRecord.illustration` (`@saxonberg/types`). The wire
carries the **key**, same as it carries `longDescription` prose.

## The client: key → URL → render

`packages/client/src/config.ts` owns the base URL —
`MEDIA_BASE_URL` (overridable via `VITE_MEDIA_BASE_URL`, default the S3
bucket) and `mediaUrl(key)`. **Server emits keys; the client prepends the
base** — one contract for every surface:

- `InspectionPane.tsx` `renderSingle` — `<img src={mediaUrl(detail.illustration)}>`
  at the top of the body, `onError` hide, `alt` from `shortDescription`.
- `CharGenStage.tsx` — the race picker's 3:4 portrait slot renders
  `mediaUrl(option.image)`.

## Provenance: the `MediaAsset` Document

`lib/media/MediaAsset.ts` — a plain `Document` (collection `media_assets`),
**not a Stuff and not on the render path**. The game shows a picture from
the key on the content; this is a parallel record *about* the asset,
joined by the same key. Written by the generation pipeline; read only by
authoring / staleness tooling — the runtime render never touches it.

It earns its place once you need to *manage* the library rather than show
one image: `sourceContentHash` (sha256 of the model-derived prompt) makes
drift detectable when the model an image was rendered from changes;
`styleVersion` lets a style-bible bump sweep "regenerate everything older
than vN"; `prompt` + `model` + `size`/`quality` make a regenerate
deterministic. All fields are scalars (no marshaller wired).

## Generation: model-driven, one house style

Images are generated **from the internal model**, not hand-written
prompts — the same composed state the `look` pipeline reads (a location's
description + contents + exits + door state; a species' description +
bodyplan + material). The image is a render of the standard-model.

- **Style**: one locked house aesthetic (Beatrix-Potter / E.H.-Shepard
  watercolor) shared verbatim by every generation, so the library reads
  as one book rather than variable AI output.
- **Provider**: `gpt-image-1` (no Anthropic image model). `OPENAI_API_KEY`
  in `packages/server/.env`.
- **Serving**: S3 bucket `panterasbox-media` (us-west-2, public-read).
  Content stores the key; the client builds the URL.

### The pipeline (offline tooling)

Not part of the server runtime. `packages/server/src/tools/illustrate.ts`
(via `illustrate-preload.js`, which registers the call-security loader
hook first) boots the engine against an isolated scratch DB
(`<db>_media`), resolves a content Stuff, builds the model-derived prompt,
generates, uploads to the content's own key via the `aws` CLI (keeping the
AWS SDK out of the server deps), and upserts the `MediaAsset` record.
`ILLUSTRATE_SKIP_IMAGE=1` does a record-only backfill. `tools/imagegen/`
holds the earlier static (seed-reading) harness. Generated PNGs live only
in S3 — never in git (`tools/imagegen/samples/` is gitignored).

## Char-gen integration

The race picker shows a species portrait + a structured **dossier**. Both
are *presentation derived from the species model*, so they live in
`SpeciesApi.buildDossier(species, path)` (`api/species.ts`), not the
controller. `EnrollController` pre-warms a per-species card (`{ dossier,
illustration }`) — the dossier from `SpeciesApi`, the illustration from
`species.getIllustration()` — and surfaces it on `CharGenOption`. See
[char-gen.md](./char-gen.md).

## Deferred

- **Recognition / disguise gating.** An illustration is viewer-perceived
  presentation and should eventually ride recognition's
  `getPresentation()` (a hooded figure shouldn't flash their portrait).
  v1 ships it as a flat projected field — safe, because v1 content (rooms,
  species-*kind* portraits) doesn't disguise. Retrofit when recognition
  lands and per-instance NPC portraits exist. See
  recognition slate.
- **Cohesion post-process** — a shared palette LUT + paper-grain pass over
  every generated image. Trigger: only if generated cohesion visibly
  drifts at library scale.
- **Staleness audit** — a tool that recomputes a content item's
  `sourceContentHash` and flags images whose model drifted.
- **External embeds** — Twitch / third-party video as `Embed` reference
  value-types + an embeddable mixin (the second flavor above).
- **CloudFront + custom domain**; **in-CMS on-demand generation + a quota
  system** (per-author credit pool — the only place generation cost needs
  metering; offline generation is one-time and self-controlled).
