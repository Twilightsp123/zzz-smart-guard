> **SUPERSEDED / REJECTED AS PRIMARY DESIGN (2026-11-02):** A user must be able to issue an RMB move and have it take effect immediately **without first disabling a hold-position toggle**. The native `Can't Move` + 100% speed debuff architecture violates that requirement. Keep this document as a comparison/technical reference only. **New active design:** [RMB_FIRST_DESIGN.md](RMB_FIRST_DESIGN.md).

# Smart Guard v2.2 research — native Hold Position ability

Historical proposal only. **Not approved for implementation**: violates seamless RMB movement; no build or gameplay validation. Keep v2.1 GitHub Release untouched.

## Recommendation

Build a separate DB-driven, player-toggleable **Hold Position** ability for ranged units. When enabled it applies BOTH:
- the ability phase **Can't Move** property, and
- a **100% reduction of movement speed** for the phase.

The unit should keep normal ranged firing decisions while being unable to reposition for LOS, range, and firing arcs. A manual ON/OFF toggle makes it straightforward for players to reposition when needed. The system avoids Lua position polling, repeated `halt()`, and `release_control()`.

Existing working precedent: Snek's *Hold Position – Hold Ranged Units Stationary (9.0.1)*, which describes this exact mechanism and says individual models/pieces pivot to fire in place:
https://steamcommunity.com/workshop/filedetails/?id=2869716515

## Concrete DB table path (verify with current WH3 RPFM schema)

1. `unit_abilities_tables` — unique ability key, UI/icon/localization reference.
2. `unit_special_abilities_tables` — active/toggle behavior and unique numeric ID. **Copy a relevant current-game toggleable ability row** rather than guessing fields/flags.
3. `special_ability_phases_tables` — new phase with **Can't Move** enabled, duration/transition matching the toggle.
4. `special_ability_phase_stat_effects_tables` — apply movement speed reduction while this phase is active. **Do not guess the current stat key or numeric sign/unit**: inspect a proven equivalent record and WH3 9.x schema.
5. `special_ability_to_special_ability_phase_junctions_tables` — link ability and phase.
6. `land_units_to_unit_abilities_junctions_tables` — attach to appropriate ranged land units for custom battles, **only existing keys in the current game DB**.
7. `text/db/*loc` — English/Chinese strings; icon can be temporary but must not reference missing assets.

All key names must be namespaced to this mod, not copied as duplicates from another mod. Use RPFM Schema for the **current executable version** and explicit referential-integrity checks before packing.

Mechanics/background:
https://steamcommunity.com/sharedfiles/filedetails/?id=1698960734

## Why not use only Lua?

- Guard mode is `is_behaviour_active("defend")`. It does NOT guarantee ranged units will not chase for line of sight (CA issue report): https://community.creative-assembly.com/total-war/total-war-warhammer/bugs/5212-hold-position-for-ranged-units
- `battle_unit:set_stat_attribute()` documents a fixed list of keys; **'cant_move' is not documented as a valid Lua attribute**. Do not blindly invoke `set_stat_attribute("cant_move", true)` and claim success. The ability-phase **Can't Move** DB field *is* documented.
- A pure-Lua attack callback plus 100ms position polling cannot guarantee no movement occurs, nor can it perfectly identify who issued a move.
- `ordered_position()` is not a pre-chase oracle: during attacks it returns current location.
- Command handlers in community mods do not reliably identify the affected unit or the source of an order; automatic enable/disable on player Move remains an unresolved UX/integration problem.

## Scope/behavior decisions

- **Manual toggle is the first implementable target**. Automatically activating on Guard Mode and deactivating on intentional Move is an OPTIONAL later feature after reliable per-unit player-order identification exists.
- Default ability should be OFF to avoid preventing retreats/repositioning.
- Don't auto-apply to AI units, or silently alter standard Guard Mode, until the UX, AI impact and gameplay behavior are verified.
- First prototype: ONE vanilla ranged infantry unit and ONE artillery unit in custom battles. Expand coverage only after testing.
- Campaign coverage and modded-unit coverage are separate: Snek describes effect-based campaign coverage, but schema details must be independently verified before copying that pattern.
- Maintain backward compatibility: `v2.1` remains available and untouched, `v2.2` is an experimental different architecture. Do not install both simultaneously unless verified.

## Known trade-offs of native lock

- While ON, manually ordered movement, repositioning and some ground targeting may be blocked. Player must toggle OFF first.
- No line of sight? Unit **stays put and may not fire**; that is intended, not a bug.
- Out of range? Cannot step into range, sometimes cannot issue a target order; document it.
- Artillery and manual attack-ground can behave differently; user report of inability to manual target with Hold enabled: https://steamcommunity.com/sharedfiles/filedetails/?id=2869716515&searchtext=hold+position
- Future WH3 patches change `land_units` keys/relationships. Community reported invalid junction record causing DB startup crash before 9.x update: https://steamcommunity.com/workshop/filedetails/?id=2869716515
- Mechanical interactions with charge, rout, skirmish, flying units, weapon teams, mounts and move-fire units require separate tests.

## Do not ship yet

Required inputs to create a valid experimental `.pack`:
- Current version of WH3 vanilla DB tables and schema (from local RPFM).
- Reference **current version** ability rows or a known compatible toggle ability.
- Valid list of target units and their land_unit keys.
- In-game test/log evidence for core acceptance cases.

Next prototype should contain *only* native ability & DB rows in isolated pack; no automatic Lua command intervention. Keep the Lua v2.1 package as fallback/legacy until comparisons pass.
