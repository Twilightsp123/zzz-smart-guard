# Smart Guard v2.1 — static source audit (2026-11-02)

Status: **source verified, engine behavior not tested**. Original `zzz_smart_guard.pack` remains unchanged in release v2.1.

## Verified current implementation

Internal path: `script\\battle\\mod\\smart_guard.lua`, 277 lines. The file is stored inside the existing release archive.

- Lines 72-151: `BattleUnitAttacksEnemyUnit` handler. It selects player-controlled units with ammunition and `is_behaviour_active("defend")`. It records attacker center coordinates and target's UI ID.
- Lines 110-114: if repeated attack event has the same target ID it retains the existing anchor.
- Lines 132-149: if target is beyond reported missile range, calls `halt_and_release` immediately.
- Lines 153-246: `bm:repeat_callback` at 100ms; skips units not moving; compares *unit center* position to the saved anchor.
- Lines 211-229: <=0.4m tolerated, >1.2m may halt without range check, 0.4-1.2m range check.
- Lines 233-239: 800ms halt throttle.
- Lines 60-70: `sunit.uc:halt()` and `sunit.uc:release_control()` are called in the same protected Lua function.
- Line 13: `ENABLE_LOGGING = false`, despite header comment claiming logging is enabled.

## Risks and flaws

1. **Reactive by construction**: a unit needs to start moving and displace its center before this monitor catches it. Tight polling cannot provide guaranteed pre-movement prevention.
2. **Player intent is ambiguous**: a non-firing movement or other mod-issued order can be confused with attack-related repositioning, depending on changes to `current_target`. There is no reliable distinction between manual moves and auto-reposition.
3. **Order loss**: `halt()` may interrupt the attack order, causing a halt-and-reissue loop or loss of the target. `release_control()` does not restore the prior order.
4. **Target switching bypass**: changing target resets the anchor, allowing a new position to become the anchor after drifting.
5. **Incomplete unit validity**: `number_of_men_alive()>0` and not routing is not a general guarantee that every engine method will succeed on a unit pointer.
6. **Error observability**: widespread silent `pcall` means failures are hidden; an error in `uc:halt()` prevents reaching `release_control()` in that same block. There is no proof `pcall` can catch native access violations.
7. **Target and geometry**: `unit_distance` measures bounding box gap, not a line-of-sight or firing-arc check. `missile_range` alone cannot say a unit can shoot from its current spot.
8. **Center position**: `position()` is the center of the main squad. Internal soldier shuffling may be missed while center remains stable.
9. **Unwanted idle polling**: single 100ms monitor callback continues for the battle even if no anchors; its scale is minor for limited units but unnecessary when the core behavior can be prevented in DB.

## API evidence

- https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_unit.html — `position()`, `ordered_position()`, `unit_distance()`, behaviors.
- https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_unitcontroller.html — `halt()`, `release_control()`.
- `ordered_position()` returns the **current** position when a unit is attacking; cannot be used alone to preempt all attack-induced pathfinding.
- https://steamcommunity.com/workshop/filedetails/discussion/3082390369/4027970145567187253/ — reported limitations of battle command handlers for distinguishing player/script orders.

Do not claim the above are experimentally confirmed bugs in every combat state. This is a static audit and test hypothesis list.
