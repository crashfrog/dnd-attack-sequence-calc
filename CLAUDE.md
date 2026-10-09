# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A D&D attack sequence calculator that rolls multiple attacks and shows cumulative damage at each AC threshold. Single-file React app (`index.html`) using CDN-loaded dependencies (React 18.2, Babel standalone, Tailwind CDN, Three.js r128). Only `index.html`, `LICENSE`, and this file exist; there are no tests, linter, or package manifest.

## Development

Open `index.html` directly in a browser. No build step. The JSX is compiled in-browser by Babel (`<script type="text/babel">`), so syntax errors surface only in the browser console. CDN access is required to run.

## Status: 3D dice animation is deliberately disabled

Commit `80f939b` disabled the dice animation; do not "fix" or re-enable it unless asked.

- `DicePhysicsEngine` (Three.js, collision physics) and the `#dice-canvas` element are still present, and `diceEngine.init()` still runs on load via `setTimeout`. Nothing ever calls the engine to roll or show dice.
- `rollDie()` still pushes `{sides, value, color}` into a `collector` array, and `performAttackSequence` builds a `diceCollector` that is discarded. The color helpers (`getDamageColor`, `getSmiteColor`, `getBrutalColor`, the d20 color in `rollD20`) exist only to feed that path.
- The `labelPrefix` parameters on the roll helpers are likewise unused by anything visible.
- Treat all of the above as dormant code. Keep it intact when editing nearby logic, and keep roll helper signatures stable so the animation can be reconnected.

## Architecture

Everything lives in `index.html`. Locate code by name, not line number (line numbers drift):

- `DicePhysicsEngine` class: dormant 3D dice engine (see above).
- Inline SVG icon components (`Swords`, `Settings`, `Dices`, `Plus`, `User`, ...).
- `DnDAttackRoller` component: all app state, roll logic, and UI.
- `App`: thin wrapper, mounted with `ReactDOM.render` (React 18 legacy root API).

Roll logic is defined as closures inside `DnDAttackRoller`, not as pure module-level functions, so it cannot be unit-tested without extracting it.

**State:** `characters`, `activeCharIndex`, `showConfig`, `results`, `attackMode` (global: normal/advantage/disadvantage), `showTypedDamageBreakdown`, `xValue`. Only `characters` is persisted.

**Data model** (localStorage key `dnd-characters`, saved on every change to `characters`):
- Character: `name`, `elvenAdvantage`, `savageAttacker`, `attackTypes[]`.
- Attack type: `name`, `count`, `bonus`, `damageDice`, `critRange`, `smiteDice`, `brutalCritDice`, `advantage`.
- The load effect migrates older shapes: a flat pre-`attackTypes` character becomes one attack type, `critExtraDice` becomes `brutalCritDice`, and missing flags default to `false`. Add a default there when adding a new field, or old saves will have it `undefined`.
- The default character is duplicated in two places in the load effect (no saved data, and JSON parse failure). Keep them in sync.
- `count`, `damageDice`, `smiteDice`, `brutalCritDice` and names are strings that may contain `{x}` expressions; `bonus` and `critRange` are numbers.

**X-value scaling** (`resolveX`):
- `{x}` resolves to the current X value (the stepper in the UI).
- Arithmetic: `{x+2}`, `{x-1}`, `{x*2}`, `{x/2}`. Division is floored and `/0` yields 0. The operand must be a literal integer.
- Step scaling: `{x:start/step}` gives `max(0, floor((x-start)/step)+1)`, for spell slot progressions. `step <= 0` yields 0.
- Resolved in attack `count`, all damage expressions, and character/attack names (display).

**Dice and typed damage:**
- `parseDice` splits on `+` only. It does not support subtraction (`1d8-1`), so negative modifiers parse incorrectly. `{x-1}` works because it resolves to a number before parsing.
- Damage expressions support type annotations: `2d6(fire)+1d8(cold)`.
- `parseTypedDamageExpression` returns `segments` (dice, flat, type) and `firstSpecifiedType`. Untyped segments are untyped on their own, but brutal-crit and smite rolls take the main damage's `firstSpecifiedType` as `fallbackType`.
- `mergeDamageTypes(target, source, multiplier)` aggregates by type, deletes zeroed keys, and is used with `-1` to subtract attacks when computing thresholds.
- `expectedValueOfExpression` computes mean damage (used by Savage Attacker).

**Attack resolution** (`performAttackSequence`):
- Effective mode comes from `getEffectiveAttackMode(attack.advantage, global attackMode, char.elvenAdvantage)`. Advantage plus disadvantage cancels to normal. Advantage with the character's `elvenAdvantage` becomes `elven` (3d20 keep highest). Per-attack `advantage` applies regardless of the global mode.
- Natural 1 is an automatic miss (excluded from thresholds). Otherwise there is no AC comparison at roll time: each attack's total (`d20 + bonus`) is the AC it just reaches. A natural 20 is flagged `isNat20` and displayed as an auto-hit.
- Crit = `d20 >= critRange`. A crit rolls `damageDice` a second time (so flat modifiers are doubled too), plus `brutalCritDice` on every crit, plus `smiteDice` on the first crit of the whole sequence only (`smiteApplied` spans all attack types).
- Savage Attacker (character-level): once per sequence, on a hit with d20 > 10 whose main damage is below its expected value, reroll the main damage and keep the higher result. Crit and extra dice are unaffected.
- Attacks sort ascending by total, misses last. Thresholds walk that list: the damage at AC `n` is the cumulative damage of every attack with total >= `n`, built by subtracting each attack's damage as the walk moves up.
- `formatOutput` builds the one-line summary string (`N damage (typed breakdown) if AC hits [attack name] (crit)`). `showTypedDamageBreakdown` toggles the typed portion.
- The results screen also shows each attack's raw d20 rolls, crit flag, and Savage Attacker reroll detail.
