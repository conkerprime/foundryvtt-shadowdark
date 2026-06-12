# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A community-maintained Foundry VTT game system for the Shadowdark RPG. The output is `system/shadowdark-compiled.mjs` (a Rollup bundle), along with compiled CSS, i18n JSON, and LevelDB compendium packs — all served by Foundry VTT from the `system/` directory.

## Commands

```bash
npm install               # install dependencies

npm run build:watch       # compile JS/CSS/i18n + watch (no packs) — use during development
npm run build             # full build including compendium packs (run before committing pack changes)
npm run lint              # ESLint with auto-fix
npm run notes             # compile RELEASE_NOTES.md into a journal compendium pack

npm run export            # export LevelDB packs → JSON source files in data/packs/
npm run import            # compile JSON source files in data/packs/ → LevelDB packs in system/packs/
npm run lang              # compile i18n/**.yaml → system/i18n/*.json
npm run css               # compile SCSS → system/css/shadowdark.css
```

There is no test suite.

## Development Setup

Requires Node.js v22.x.

1. Copy `example-foundry-config.yaml` to `foundry-config.yaml` and set `installPath` to your Foundry VTT installation directory.
2. Run `npm run createSymlinks` — creates a `foundry/` directory with symlinks to Foundry's `client/`, `common/`, and lang files, enabling IDE IntelliSense for Foundry APIs.
3. Symlink the `system/` directory into Foundry's user data `systems/` folder:
   - **Linux/macOS**: `ln -s <repo>/system <foundry_data>/systems/shadowdark`
   - **Windows**: `mklink /d %localappdata%\FoundryVTT\Data\systems\shadowdark <repo>\system`
4. Run `npm run build:watch` during development. Start Foundry with `--hotReload` for automatic CSS/i18n reloading, or pair with the LiveReload Chrome extension for full page reloads.

## Architecture

### Entry Point and Module Namespace

`system/shadowdark.mjs` is the source entry point. Rollup bundles it to `system/shadowdark-compiled.mjs` (declared in `system.json`). On `init`, the system exposes everything via `globalThis.shadowdark` / `game.shadowdark`.

### Layer Breakdown

| Directory | Purpose |
|---|---|
| `system/src/documents/` | Foundry document subclasses: `ActorSD`, `ItemSD`, `ActiveEffectSD`, `ChatMessageSD` |
| `system/src/models/` | TypedDataModels registered with Foundry's data model system — `PlayerSD`, `NpcSD`, and one file per item type under `models/items/` |
| `system/src/sheets/` | Actor/item sheet ApplicationV1 classes: `PlayerSheetSD`, `NpcSheetSD`, `ItemSheetSD`, `LightSheetSD` |
| `system/src/apps/` | Standalone UI applications (character generator, level-up wizard, light source tracker, importers, roll dialog, etc.) |
| `system/src/hooks/` | Foundry hook listeners, grouped by concern; wired up in `hooks.mjs` |
| `system/src/dice/` | `RollSD` extends `foundry.dice.Roll` with crit/success logic and custom chat templates |
| `system/src/chat/` | Chat message rendering helpers |
| `system/src/migrations/` | `MigrationRunnerSD` runs `UpdateBaseSD` subclasses on world load; each migration file is `Update_YYMMDD_N.mjs` with a numeric `version` |
| `system/src/config.mjs` | `SHADOWDARK` constants object — registered as `CONFIG.SHADOWDARK` |
| `system/src/settings.mjs` | System settings registration |

`_module.mjs` files in each directory re-export everything from that folder and are what `shadowdark.mjs` imports.

### Templates and i18n

Handlebars templates live in `system/templates/`. i18n source files are YAML in `i18n/` and compiled to JSON in `system/i18n/` — **edit the YAML, not the JSON**.

### Compendium Packs

Source data lives as JSON in `data/packs/<pack-name>/`. The build step compiles these to LevelDB format in `system/packs/`. To edit pack content: run `npm run export` to get JSON, make changes in `data/packs/`, then `npm run import` (or `npm run build`) to recompile. Never edit LevelDB files directly.

## Code Conventions

- All system classes are suffixed `SD` (e.g., `ActorSD`, `RollSD`, `LightSourceTrackerSD`).
- **Indentation**: tabs (not spaces). **Quotes**: double. **Brace style**: stroustrup. **Max line length**: 100.
- Lint is enforced via ESLint with auto-fix (`npm run lint`). CI runs on every push/PR to `develop`.
- i18n keys follow the pattern `SHADOWDARK.category.key` and must be added to `i18n/en.yaml` first.
- Migrations must extend `UpdateBaseSD`, set a static `version` as a numeric `YYMMDD.N`, and be exported from `migrations/updates/_module.mjs`.

## Shadowdark-Specific Hooks

The system emits custom hooks via `Hooks.call` (cancellable — returning `false` aborts the action) or `Hooks.callAll` (non-cancellable). All hooks are documented at the [wiki Hooks page](https://github.com/Muttley/foundryvtt-shadowdark/wiki/Hooks).

| Hook | Cancellable | Fires | Argument |
|---|---|---|---|
| `SD-Roll-Dialog` | No | Before roll dialog renders | `context` — render context; modify to add situational effects or ammo choices |
| `SD-Stat-Check` | Yes | After dialog confirm, before ability check roll | `context` — roll config object |
| `SD-Player-Attack` | Yes | After dialog confirm, before player attack roll | `context` — roll config object |
| `SD-Player-Spell` | Yes | After dialog confirm, before player spell cast | `context` — roll config object |
| `SD-NPC-Attack` | Yes | After dialog confirm, before NPC attack roll | `context` — roll config object |
| `SD-NPC-Spell-Cast` | Yes | After dialog confirm, before NPC spell cast | `context` — roll config object |
| `SD-Player-classAbility` | Yes | After dialog confirm, before class ability use | `context` — roll config object |

`SD-Stat-Check` applies to both PCs and NPCs. The roll config object passed to all cancellable hooks shares the structure described in the Rolls section below.

## Roll Config Object

Rolls are driven by a `rollConfig` object assembled before the dialog and passed to hooks. Key fields:

```js
{
  // Required
  actorId,          // Actor UUID

  // Optional context
  itemUuid,         // UUID of weapon/spell/item being used
  targetUuid,       // UUID of targeted token

  // Presentation
  type,             // "check" | "attack" | "spell" | "ability"
  heading, title,   // localised display strings
  rollMode,         // "public" | "private" | "blind" | "self"
  skipPrompt,       // boolean — bypass dialog entirely
  descriptions,     // string[] of HTML shown on the roll card

  // Effect management
  situational,      // UUIDs of available situational effects
  selected,         // UUIDs chosen by the user in the dialog

  // Per-roll options (mainRoll / damageRoll)
  rollOptions: {
    base,           // foundation die ("d20", etc.)
    formula,        // full dice expression
    bonus,          // formatted modifier string
    dc,             // difficulty class
    canCritical,    // boolean
    criticalSuccessAt, criticalFailureAt,
    advantage,      // 1 = advantage, 0 = normal, -1 = disadvantage
    // damage-only
    criticalHit, criticalMultiplier,
  },

  // Type-specific extras
  // check:  ability ("str"|"dex"|"con"|"int"|"wis"|"cha")
  // attack: attackType, handedness, range, ammoClass
  // cast:   spellUuid, castingAbility, focus, range, duration, damageType
}
```

## Data Model Reference

### Player (`system.*`)

| Path | Type | Notes |
|---|---|---|
| `abilities.<str\|dex\|con\|int\|wis\|cha>.value` | int | |
| `abilities.<…>.mod` | int | read-only, derived |
| `alignment` | string | `"lawful"` \| `"neutral"` \| `"chaotic"` |
| `attributes.ac.value` | int | total AC |
| `attributes.ac.<this\|armor_base_type\|name_of_property\|unarmored>` | int | per-source AC |
| `attributes.hp.max` / `.value` | int | |
| `coins.<gp\|sp\|cp>` | int | |
| `level.value` / `.xp` | int | |
| `luck.available` / `.remaining` | int | |
| `renown` | int | |
| `slots` | int | gear slot maximum |
| `slotUsage.<coins\|gear\|gems\|treasure\|total>` | int | read-only |
| `languages` | UUID[] | |
| `spellcasting.classes` | string[] | |
| `ancestry`, `background`, `class`, `deity`, `patron` | UUID | read-only references |

### NPC (`system.*`)

| Path | Type |
|---|---|
| `abilities.<str\|dex\|con\|int\|wis\|cha>.value` / `.mod` | int |
| `attributes.hp.max` / `.value` | int |
| `attributes.ac.value` | int |

## Active Effects

Active Effects modify actor data by targeting `system.*` paths. In AE formulas, drop the `system` prefix and use `@` notation (e.g., `@attributes.hp.max`). Functions `ceil()`, `floor()`, `min()`, `max()` are supported.

**Common change modes:** `Add` (standard modifier), `Multiply` (e.g., backstab dice), `Override` (replaces value; default priority 50).

**Attack modifier path pattern:**
```
system.roll.attack|melee|ranged.bonus|damage|advantage|critical-multiplier.all|this|<weapon-type>
```
`all` applies to every weapon; `this` applies only to the specific weapon carrying the effect.

**Situational effects** are flagged as conditional and can be toggled on/off per roll in the roll dialog. The AE Panel's predefined-effects dropdown pre-fills attribute keys for common class/ancestry/talent effects.

## Foundry VTT API Patterns

The full API reference is at https://foundryvtt.com/api/. Key patterns used throughout this codebase:

### DataModel (`foundry.abstract.DataModel`)

Custom data lives in TypedDataModels registered in `CONFIG.Actor.dataModels` / `CONFIG.Item.dataModels`. Key overrides:

- `static defineSchema()` — returns a `DataSchema` using `foundry.data.fields.*` field types
- `static migrateData(source)` — transforms legacy source data on load; use `_addDataFieldMigration` for simple renames
- `prepareDerivedData()` — computes derived values after base data is set; do not mutate `_source` here
- `validateJoint(data)` — cross-field validation after individual field checks

**Common field types** (`const fields = foundry.data.fields`):

| Field | Use |
|---|---|
| `StringField` | Text values |
| `NumberField` | Numbers; pass `{ integer: true }` for integers, `{ min, max }` for bounds |
| `BooleanField` | True/false |
| `SchemaField` | Nested object with its own field definitions |
| `ArrayField` | Ordered list; wrap the element type: `new ArrayField(new StringField())` |
| `DocumentUUIDField` | UUID reference to another document |
| `HTMLField` | Rich text / HTML content |
| `ObjectField` | Arbitrary key-value object |

### Document Lifecycle (`foundry.documents.Actor` / `foundry.documents.Item`)

Pre-operation methods run only on the requesting client; post-operation methods run on all clients.

| Method | Side | Return `false` to cancel? |
|---|---|---|
| `_preCreate(data, options, user)` | requesting client | yes |
| `_preUpdate(changed, options, user)` | requesting client | yes |
| `_preDelete(options, user)` | requesting client | yes |
| `_onCreate(data, options, userId)` | all clients | no |
| `_onUpdate(changed, options, userId)` | all clients | no |
| `_onDelete(options, userId)` | all clients | no |

Data preparation order: `prepareBaseData()` → `prepareEmbeddedDocuments()` → `prepareDerivedData()`. Use `getRollData()` to expose data for dice formulas; never mutate the original object.

Use `this.updateSource(changes)` (not direct assignment) inside `_preCreate` / `_preUpdate` to stage changes before they're committed.

### ApplicationV2 (`foundry.applications.api.ApplicationV2`)

New apps should extend `ApplicationV2` (or `DocumentSheetV2` for sheets). Key differences from the legacy V1 pattern:

- **No `activateListeners`** — wire events declaratively via `data-action="actionName"` on HTML elements and define handlers in `static DEFAULT_OPTIONS.actions`.
- **`_prepareContext(options)`** — returns the template data object; replaces `getData()`.
- **`_onRender(context, options)`** — runs after every render; `_onFirstRender` runs only on initial render.
- Handler signature for actions: `static async myAction(event, target)`.

```js
static DEFAULT_OPTIONS = {
  actions: { myAction: MyApp.myAction },
  form: { handler: MyApp.onSubmit, submitOnChange: false },
};
```

### Hooks (`foundry.helpers.Hooks`)

- `Hooks.on(name, fn)` / `Hooks.once(name, fn)` — register a listener; returns a numeric ID.
- `Hooks.off(name, fnOrId)` — deregister.
- `Hooks.call(name, ...args)` — fires listeners sequentially; a listener returning `false` stops the chain (used for cancellable SD hooks).
- `Hooks.callAll(name, ...args)` — fires all listeners regardless of return values (used for `SD-Roll-Dialog`).
