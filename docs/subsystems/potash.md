# Potash — the ashery

Wood ash into lye, lye into black salts, black salts into pearl ash: the
glasshouse's flux and the soapmaker's alkali, made from the wood's waste.
The trade is `trade-potash` at `/trade/potash`; ash itself is `trade-fuel`'s
(`/trade/fuel/thing/Ash`), because the kernel Burner and the clamp both make
it, upstream of any ashery. **No new verb**: every act is the fire's, the
hopper's, the pot's and the kiln's.

> ⭐ **The strength comes from the wood and carries end to end.** Beech
> strong, pine weak, brash richer than heartwood; the heap's alkali share
> → the lye's kilograms per litre → the cake's mass and grade → pearl ash.
> Nothing anywhere stores a "strength" but the heap's composition and the
> liquor's payload — and no number for it ever reaches a player.

## The chain

| step | act | mechanism | where it lives |
|---|---|---|---|
| fell | `fell beech` | every felling trims **brash** (ten bundles, a fifth of the bole) of the stand entry's `brashMaterialPath` | `FellController.dropStandard` |
| burn | `stoke brash into pit`, `ignite pit`, `draught wide` | the **ash pit** is simply a big Burner (`AshPit extends Firebox`) | trade-potash |
| rake | `sweep pit` (a broom) | the grate rule: kg burnt × `ashFraction`, its alkali by `alkaliShare` → heaps of `fire.ash.heapKg` | `Burner.consumeFuel`, `mintAshHeaps` |
| leach | `put ash in hopper`, `pour pail into hopper` | the liquor PERCOLATES down through the heaps over half a day, each giving up its alkali; heaps leave **spent ash** once it is through; a lye poured back through fresh ash is a second pass | `Ash` (`PercolatingMixin`), `Hopper.leach` |
| read | `analyze density hopper` (an egg in reach) | the **egg** is the density reading's eye rung; the glass words a lye too | `DensityReading`, `LyeStrength` |
| boil | `pour pail into pot`, a fire under it | the kernel's latent-heat **plateau**; the pot keeps count of the alkali and casts the cake when dry | `IronPot` |
| calcine | `put cake in kiln`, `fire kiln` | the black salts' own `calcine` heat transition (ceramics M1 — no recipe); the same cake comes out, made of pearl ash, its potash concentrated | `ThermalMixin` heat-work |
| sell | `sell "pearl ash"` | buying lines at the glasshouse board and the chandlery counter | `Stock.buyFrom` |

## The numbers (grain — an author's)

- **Ash.** `Material.ashFraction` (beech 0.01, pine 0.005; brash ×3) and
  `Material.alkaliShare` (beech 0.18 … pine 0.08; brash ×1.3). A raked
  heap's composition is `[{ fire.ash.alkaliMaterial, alkaliKg / kg }]`;
  `gradedByStrength` rows re-grade by it (<0.10 poor · <0.165 fair · <0.22
  fine · <0.30 exceptional) and the look line reads it: *grey and gritty* ·
  *an even grey* · *pale and fine … strong ash*.
- **Lye.** `dissolvedSolidsKgPerL` on the liquor's payload — **the sugar
  build's member**, one field for a lye's alkali and a juice's sugar. The
  leach adds `Σ(heap mass × strength) / litres × (1 − 0.04 × °dH)` to what a
  poured-back lye carried. Bands (`potash.lye.*`): water < 0.005 ≤ weak <
  0.03 ≤ soap strength < 0.06 ≤ strong. An unmeasured lye (a flask off a
  shelf) reads its material's own `solidsKgPerL` (lye and caustic lye
  author 0.04, soap strength) — the constant retired with
  `potash-on-sugar`.
- **The egg.** It sinks in water; stands on its point in a weak lye (a
  novice cannot tell that from water — *it sinks*); shows a coin of shell at
  soap strength; rides high when strong. A kept, blown test egg — a `Tool`
  with `flotation`, not the hen's egg (food is not a tool).
- **The boil.** Lye authors `latentHeatOfVaporization 2260000`, so it holds
  at the boil and loses water instead of flashing empty. Cake = alkali ÷
  0.65 purity, in the pot's shape, graded as the lye was.
- **Pearl ash.** `calcine` on the black-salts row: 900 K, `massRetained
  0.8` — the purity concentrates 0.65 → ~0.81 (a heat transition keeps its
  constituents in less mass), and the cake keeps its grade and its owner
  because it is the same cake. `pearl-ash` is a MATERIAL whose `castTemplate`
  is the pearl-ash row, so a counter listing that row buys the calcined cake
  as pearl ash (`Stock.lineKeyFor`, the ceramics × potash merge).

## Tags

`flux` is the **glassmaker's** — wood ash, potash, black salts, pearl ash
(and soda, when the kelp burn ships). The glass batch's lime slot asks
`stabilizer` (quicklime), so a lye — alkali and caustic, but no stabilizer —
fills no slot. Limestone's smelting role is `smelting-flux`; `burn-lime`
burns a `carbonate`. `alkali` stays the cleansing family.

## Lime in the soap kettle

Water poured on quicklime in the kettle **slakes** it: the lump goes, its
heat (1.15 MJ/kg, `chandlery.lime.slakingJPerKg`) goes into the kettle, the
interior is **milk of lime**, and a bare hand is burnt. Lye and milk of lime
make **caustic lye**; fat and caustic lye make a hard-soap paste that boils
straight to the hard curd the frame sets — no salting. The salting-out route
is untouched.

## The places

- **The Hanging Wood ashery** (`rejection`, persistent), west of the oak
  clearing: the pit, its own kiln, a hearth with **two** iron pots (a strong
  and a weak lye on one fire), a hopper, a rain butt (1 °dH), the test egg, a
  pail, a broom, a store of six heaps of wood ash, four cordwood, the
  ash-burner. The ride now stands beech and pine beside the oak and ash.
- **The glasshouse** buys pearl ash (20) and black salts (12) on a chalked
  board; its stock of clamp-ash became wood ash.
- **The Wharfside chandlery** (still vacant — a player-shaped hole) buys
  wood ash (3), black salts (12) and pearl ash (20), and props a rain butt
  and an egg.
- **The ash round** — a sole trader's Business of one (the tinker's shape,
  no operating locations): the `ashes` brain rakes the grates on his stops,
  pays each heap's owner the chandlery's own bid from his own account, takes
  title, and sells at the counter with `sell`. He earns by volume, never by
  margin.

## Whose it is — the communal oven

The chain's goods are titled by the kernel's fresh-good rule
([chattel.md](./chattel.md)), extended by this build:

- **A liquid carries its owner** (`BulkPayload.contentsOwner`). The hopper
  writes it from the heaps' titles; a pour blends it by agreement: untitled
  water makes no claim, and two owners make it nobody's.
- ⭐ **What is made in somebody else's vessel belongs to whoever owned the
  contents.** Your lye in the ashery's pot makes your cake. When nothing
  titled went in (the ashery's own ash store, rain), it belongs to the
  **maker**, the person whose pour or put the leach was part of.
- **The pot's cake is a process mint**: it sets when somebody reads the pot,
  so it declares its maker from the lye, and the onlooker is never titled.
- **The kiln's pearl ash is made from its charge**. When the charge was
  nobody's, it belongs to whoever fired the kiln, carried across the firing's
  completion as the continuation's author.

## Limits and seams

- **The boil runs on read.** The kernel plateau reconciles when someone
  reads the pot, keeping the alkali as the water leaves; the pot reads the
  strength the plateau wrote and casts the black salts at saturation. It
  keeps no count of its own.
  ⚠ A long unobserved stretch is ONE step of the plateau: a pot left 24 h on a
  burning hearth barely boils, while read hourly it sets in hours (the live
  walk, 2026-10-22). This belongs to silk's plateau, and the drive reads
  hourly.
- **A hopper drawn early** gives water carrying the alkali the egg already
  reads (the leach re-keys the liquid to lye later). The pot counts dissolved
  solids in any liquor, so nothing is lost.
- **The ash round** stands up where its hand is cast and at every stop
  (`LocationGraphRegistry`), and the roster fetches the ash-man to a stop
  someone is standing in (`EmploymentLogic`). On a fresh world he takes little
  ash, because the town's hearths have burned little: *a supplement, not a
  living* (AC 9).

- ✅ **Ash percolates (cordage W7, 2026-10-30).** `Ash` composes the kernel
  `PercolatingMixin`: water poured into the hopper is put on the heaps,
  works down through them over half a game day, and each heap gives up its
  alkali as it goes — spent ash only once the water is through. The hopper
  decides only whose the lye is, at the fill.
- ✅ **`potash-on-sugar` landed with the sugar build (MR !439, 2026-10-24).**
  Potash's `dissolvedSolidsKgPerL` declaration and its lye-only fold in
  `BulkableLogic` are gone (a lye's strength is the same fold as a juice's
  sugar); lye authors `solidsKgPerL` (replacing `UNMEASURED_LYE_KG_PER_L`);
  `IronPot`'s `alkaliKg`/`lastSeenL` count and its re-base are gone — the pot
  reads the plateau and casts at saturation. The percolation closed with
  cordage W7 (above); the drive's equal-potash TIMING checkpoint is the
  cordage drive's.
- Rain spoiling open ash, big ash (burnt ground), spent ash on a field, the
  kelp burn, a fresh hen's egg as a hand tool, a solid put into a filled vat
  re-matching, person-to-person purchase, river freight — see the potash
  slate and the plan's deferred seams.
- ⚠ The glass melt is **ungraded** (the glass trade grades by colour, never a
  band), so the strength that reaches glass rides only the melt's
  composition.

## History

- **2026-10-19 — the potash build** (`reqs/potash`, W0–W7): ash sized and
  strengthened by the wood; brash; the pit, the hopper's derivation, the egg;
  `soda-from-lye` retired (boiled wood-ash lye is potash, never soda); the
  iron pot on silk's plateau; pearl ash by `fire`, the firing's yield
  concentration and weakest-input grade; the flux/stabilizer retag; slaking
  and caustic lye; the ashery, the buying lines and the ash round. See the
  potash plan's drive record for what driving found.
- **2026-10-21/22 — verification** (`a8e9a84ad`…`717476710`): 21 drive runs
  and a live browser walk found the cast at saturation, the transfer's
  source and destination reconciles, the pour hand-over between two views of
  one verb, the round that never ran, the full crock, and the attribution
  build (continuations carry their author; the communal oven). See the
  plan's drive record.
