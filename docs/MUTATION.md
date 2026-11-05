# Mutation testing of the conformance suite

`scripts/crocus-mutation.mjs` measures how many small, single-point bugs the black-box conformance
suite (`crocus/benchmark/conformance/`, docs/CONTRACT.md) catches. It complements the mutant
calibration adapter (docs/BENCHMARK.md): that one injects many bugs at once and looks at the score;
this one injects **one** bug at a time and asks, per module, which bugs the suite can see.

```bash
node scripts/crocus-mutation.mjs --work /tmp/mut --per-module 15 --jobs 4 --md docs/MUTATION.md   # full campaign (~3 h on 4 cores)
node scripts/crocus-mutation.mjs --work /tmp/mut --modules core,time --per-module 6                 # a quick look
node scripts/crocus-mutation.mjs --work /tmp/mut --rerun results.json --fp-first                    # re-test survivors after adding tests
node scripts/crocus-mutation.mjs --list --modules physics                                           # print the mutants only
node scripts/crocus-mutation.mjs --work /tmp/mut --classify-noise results.json                       # survivors within seed noise
node scripts/crocus-mutation.mjs --report after.json --before before.json --md docs/MUTATION.md     # table only
```

## Method

**Modules.** A module is a directory `src/<area>/` with files reachable from the contract modules
(`src/game.js`, `src/time/index.js`, `src/parallel/index.js` and the tile worker). `runtime`, `ui` and
`render` are browser-only (graded by the e2e scenarios), so they are not mutated here.

**Mutants.** Every candidate is a single textual change in code (strings, comments and regexes are
masked by the benchmark's lexer), drawn from twelve operator classes:

| class | change |
|---|---|
| rel | `<` ↔ `<=`, `>` ↔ `>=`, `===` ↔ `!==` |
| arith | `+` ↔ `-`, `*` ↔ `/` |
| logic | `&&` ↔ `\|\|` |
| bool | `true` ↔ `false` |
| const | a numeric literal ×0.5 or ×2 |
| offby1 | `± 1` → `∓ 1` / `± 2`, loop bounds (`<` ↔ `<=` in a `for` header), loop starts `0 → 1`, `-1` neighbour offsets → `-2` |
| call | an expression-statement call removed (`void 0;`) |
| jump | the `return` / `continue` / `break` of a one-line `if` removed |
| negate | an `if` condition negated |
| ternary | the two branches of a `?:` swapped |
| minmax | `Math.min` ↔ `Math.max` |
| provider | one field dropped from the object a state provider / serializer returns |

Per module the candidates are shuffled by a hash of (seed, file, position, class) and taken
round-robin over the classes, so each module gets a deterministic, varied sample of N mutants
(default 15; mutants that do not parse are skipped). The same seed always gives the same mutants.

**Running.** Each job owns a throwaway copy of `src/` (in `--work`). A mutant is written into the
copy, and the module's conformance files (`AREA_MAP` in the script; e.g. `geology → pressure`,
`core → every behavioural area`, plus the contract-wide `core/invariants` and
`time/continuity` files for every module) run with `GRAINFORGE_ROOT` = the copy, fastest file first,
stopping at the first failure. Each file has a timeout of max(30 s, 3 × its reference time + 10 s);
the reference must pass every mapped file first (a failing file would be excluded with a warning).

**Classification.**
- `killed`: some mapped test failed (or a file did not load).
- `timeout`: a file exceeded its timeout (counted as detected).
- `survived`: every mapped test passed, and a **differential fingerprint** differs from the reference:
  22 deterministic scenes driven only through the contract (a busy scene with every element, every
  reaction pair and every element hot / cold in compartments, presets, planets, entities, the player
  with the documented inputs, circuits and optics, weather, wipes, gravity modes, rigid bodies, every
  service, chemistry, life, Rule Lab examples, saves / rewind / slots, the creator, worldgen specs and
  2-thread parallel tiles). A differing hash means a different observable outcome.
- `equivalent?`: the tests pass and no fingerprint scene tells the mutant from the reference. These
  are *suspected* equivalent: unreachable code, behaviour only reachable through non-contract
  options, or changes too small to show in those scenes.

- `noise`: survived, but every scene's coarse observables (cells per element, living entities per
  kind) stay inside the reference's own envelope over three seed offsets (0 / 1000 / 2000, widened to
  0.85 × min − 3 … 1.15 × max + 3). Such a mutant changes the random stream (a hash constant, a
  visiting order), not what a player could measure; no tolerance-based test should reject it
  (`--classify-noise results.json`).

**Mutation score** = (killed + timeout) / (mutants − equivalent? − noise). The **strict** score
counts noise-level mutants as survivors: (killed + timeout) / (mutants − equivalent?).

A mutant that a test kills is not equivalent, so in the before / after table a "before" mutant the
fingerprint could not tell apart but a later test killed counts as a *before* survivor.

**Re-testing.** `--rerun` re-tests only the survived and equivalent mutants of an earlier result
(killed mutants stay killed: the suite only grows). With `--fp-first` an equivalent mutant is first
re-fingerprinted and runs the tests only if the (possibly richer) fingerprint now tells it apart.
`--refingerprint` reclassifies without running tests. `--suite <dir>` runs against a frozen copy of
the suite (the "before" numbers below use the suite as it was before this work).

## What the surviving mutants taught the suite

The before → after campaign added black-box tests where survivors changed real behaviour (each test
keeps the suite's rules: contract-only, tolerance-based, several seeds, no reference hashes):

| test file | property | survivors it kills |
|---|---|---|
| `chemistry/table.test.js`, `chemistry/boiling.test.js`, `life/algae.test.js` | 14 documented reagent pairs give their products; boiling dyed water leaves steam and a little pigment; algae live in water and rot on land | removed reactions, the dye residue / steam branch, the algae wet / dry branch |
| `materials/behaviours.test.js` | drying, clay firing, cement curing times, brine boiling, snow compaction, soaking, quenching; documented density / resistance / friction / viscosity orderings | the per-element update hooks (cement, snow, brine, …) |
| `time/continuity.test.js` | save mid-process → load into a fresh sim → same future, and rewind → same replay, for 14 kinds of running process | dropped state-provider fields (thermal latent heat, gas seals, structural support, forest fires, …), recorder bugs |
| `core/invariants.test.js` | frozen cells untouched by every service and wipe; no wrap-around at the world borders | missing frozen / bounds checks (combustion, wipes, thermal shock) |
| `thermal/shock.test.js` | thermal shock works from every side | a wrong neighbour index |
| `atmosphere/regions.test.js`, `atmosphere/planet-air.test.js` | weather regions are local and scale with intensity; planet air follows the local ground | region intensity clamp, a mirrored surface ray |
| `electric/discharge.test.js`, `electric/emp-scope.test.js` | lightning stays near the line between its end points; an EMP falls off with distance and leaves non-electrical cells alone | bolt jaggedness, the EMP's skip test and component check |
| `energetics/thresholds.test.js`, `energetics/antimatter.test.js` | a chemical blast does not fuse cold deuterium; fusion heat stays local; antimatter annihilates stone but not wall | the fusion heat threshold, a mirrored neighbour index, the annihilation filter |
| `entities/collision.test.js` | wide bodies collide over their whole width; a buried player never sinks through the floor | body sample count, the overlap sweep limit |
| `npcs/combat.test.js`, `npcs/farming.test.js` | soldiers shoot what they see, not through walls; villagers sow seeds on soil only | line of sight, the planting spot test |
| `creatures/{foraging,stomp,population,retaliation}.test.js` | grazing, pollinating, periodic stomps, breeding and population caps, retaliation | senses, swimming branch, stomp timer, population count, the damage hook |
| `powers/{creator,sorting,abilities}.test.js` | custom creatures wander; entropy sort layers by density along gravity; abilities land near the fight | wander timer, sort direction, the cell-index helper |
| `rules/edges.test.js`, `rules/language.test.js` | Rule Lab moves stop at the border; `spawn … radius r` paints a centred disc of empty cells; string values, `heat` amounts, error locations | out-of-bounds offsets, the disc test, string literals, heat clamps, interpolation error columns |
| `worldgen/shapes.test.js` | World Spec circles / rects / `onlyEmpty` / last-row shapes and house / lake features | the `onlyEmpty` test, shape bounds |
| `time/saves.test.js` (one test added) | custom factions and relations travel with a save | |

Finding the survivors also exposed a reference bug: `loadWorld` into a fresh sim dropped the cells
of Rule Lab elements (the UI called a separate `prepareLoad` first, which the contract does not
offer). `loadWorld` now runs a provider `prepare` hook before mapping cells by name (docs/CONTRACT.md
§10). Two reference deviations are documented rather than fixed: fire next to *frozen* water still
turns it into steam, and blasts / radiation warm frozen cells (the invariants test therefore
compares elements and registers, not temperature).

## Results

<!-- mutation-table:begin -->
Campaign: seed 1, 15 mutants per module, 2026-11-04.

| module | mutants | killed | timeout | survived | noise | equivalent? | score before | score | excl. noise before | excl. noise |
|---|---|---|---|---|---|---|---|---|---|---|
| atmosphere | 15 | 6 | 0 | 1 | 0 | 8 | 57% | **86%** | 57% | 86% |
| core | 15 | 8 | 0 | 0 | 1 | 6 | 89% | **89%** | 100% | 100% |
| creatures | 15 | 5 | 0 | 0 | 3 | 7 | 0% | **63%** | 0% | 100% |
| electric | 15 | 9 | 0 | 0 | 0 | 6 | 67% | **100%** | 67% | 100% |
| energetics | 15 | 8 | 0 | 1 | 1 | 5 | 40% | **80%** | 44% | 89% |
| entities | 15 | 2 | 0 | 0 | 1 | 12 | 0% | **67%** | 0% | 100% |
| geology | 15 | 3 | 0 | 1 | 2 | 9 | 50% | **50%** | 75% | 75% |
| life | 15 | 3 | 0 | 2 | 2 | 8 | 29% | **43%** | 40% | 60% |
| materials | 15 | 8 | 0 | 0 | 1 | 6 | 44% | **89%** | 50% | 100% |
| npcs | 15 | 6 | 1 | 0 | 2 | 6 | 56% | **78%** | 71% | 100% |
| parallel | 15 | 2 | 0 | 0 | 4 | 9 | 33% | **33%** | 100% | 100% |
| physics | 15 | 11 | 0 | 1 | 1 | 2 | 69% | **85%** | 75% | 92% |
| powers | 15 | 4 | 0 | 1 | 1 | 9 | 0% | **67%** | 0% | 80% |
| rules | 15 | 6 | 0 | 0 | 0 | 9 | 50% | **100%** | 50% | 100% |
| time | 15 | 6 | 0 | 0 | 0 | 8 | 100% | **100%** | 100% | 100% |
| worldgen | 15 | 5 | 0 | 0 | 1 | 9 | 50% | **83%** | 60% | 100% |
| **all** | 240 | 92 | 1 | 7 | 20 | 119 | 49% | **78%** | 59% | 93% |

Surviving mutants that change an observable outcome:

- `atmosphere:atmosphere/weather.js:3627:const` 32 x0.5 (line 56) (outside the seed envelope in: time)
- `energetics:energetics/elements.js:1128:const` 80 x0.5 (line 19) (outside the seed envelope in: busy, elements)
- `geology:geology/strength.js:3140:const` 500 x2 (line 67) (outside the seed envelope in: reactions)
- `life:life/plants.js:1447:jump` jump removed: return SP.vine; (line 36) (outside the seed envelope in: busy)
- `life:life/plants.js:8383:rel` === -> !== (line 211) (outside the seed envelope in: creator)
- `physics:physics/thermal.js:11272:call` call removed: ensureSettings(sim); (line 233) (outside the seed envelope in: creator, elements)
- `powers:powers/creator.js:1424:jump` jump removed: return id; (line 30) (outside the seed envelope in: creator)
<!-- mutation-table:end -->

`score` is the strict mutation score (noise-level survivors count as survivors); `excl. noise` drops
them. "Before" is the suite as it was before this work (347 tests), re-measured with the same
mutants, area map and fingerprint; "after" is the current suite (448 tests). Thirteen mutants the
noise check called noise *before* were later killed by targeted tests (t_rex stomps, breeding,
retaliation, EMP reach, border fires, body collision, the dyed-water boiling, the lye reaction,
soldiers' line of sight, entropy sort, shapes on the last row, …): aggregate counts hide real
behaviour changes, so the noise class is only a hint and the strict score is the headline.

**What still survives, and why it is left:**
- *Life* (43%): plant morphology details (the vine choice next to a climbable cell, cactus arm
  offsets, the third crown leaf, forest-region statistics). They change the shape of a plant by a
  cell or two; testing them would pin the reference's growth grammar, not a property a player sees.
- *Parallel* (33%): all four are noise: the tile RNG seed hash and redundant chunk-wake flags.
  Any thread count still gives the same world, which is what the contract asks.
- *Geology* (50%): gold's heat resistance ×2 (500 → 1000 °C, still below its melting point), and two
  noise-level ones (support direction sampled off-centre on planets, a wall-chain limit off by one).
- *Creatures / entities / npcs / powers* (63–78%): noise-level ones (bird lift on planets, flock
  cohesion, a raptor's density, the body's vertical sample spacing, the head cell of the
  line-of-sight ray, the jump-height boundary, the glitch wipe's block count) plus the creator using
  every matching ability in one decision instead of the first.
- *Physics*: without `ensureSettings` at install only the game tuning of the thermal wake threshold
  is lost (sleep timing). *Energetics*: embers live ×0.5 shorter at the low end. *Atmosphere*: half
  the weather temperature sampling rate.
