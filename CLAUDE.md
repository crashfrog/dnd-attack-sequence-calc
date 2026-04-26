# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A D&D attack sequence calculator that rolls multiple attacks, calculates damage at different AC thresholds, and displays results with 3D dice animations. Single-file React app using CDN-loaded dependencies (React 18, Babel, Tailwind, Three.js).

## Development

Open `index.html` directly in a browser. No build step required.

## Architecture

**Single HTML file** containing:
- `DicePhysicsEngine` class (lines 27-507): Three.js-based 3D dice animation with collision physics
- `DnDAttackRoller` component (lines 572-1405): Main React app handling character state, attack configuration, and dice rolling

**Data model:**
- Characters stored in localStorage as `dnd-characters`
- Each character has multiple `attackTypes`, each with: count, bonus, damageDice, critRange, smiteDice, brutalCritDice, advantage flag
- Character-level `elvenAdvantage` enables 3d20-keep-highest when character has advantage

**X-value scaling system:**
- `{x}` in any dice/name field resolves to current X value
- Arithmetic: `{x+2}`, `{x-1}`, `{x*2}`, `{x/2}`
- Step scaling: `{x:start/step}` gives `max(0, floor((x-start)/step)+1)` for spell slot progressions

**Typed damage:**
- Damage expressions support type annotations: `2d6(fire)+1d8(cold)`
- Untyped damage inherits from first specified type in the expression
- `mergeDamageTypes()` aggregates damage by type across attacks

**Attack resolution:**
- `rollD20()` handles normal/advantage/disadvantage/elven modes
- Results sorted by total attack roll (ascending)
- Thresholds show cumulative damage if that AC-or-higher hits
- Smite dice apply to first crit only; brutal critical dice apply to every crit
