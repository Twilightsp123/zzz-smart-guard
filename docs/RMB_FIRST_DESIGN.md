# Smart Guard — RMB-first design (2026-11-02)

**CURRENT PRIORITY** — This document supersedes the proposed DB `Can't Move` toggle architecture. Status: static design, NOT implemented or verified in-game. The public v2.1 release and main branch remain unchanged.

## Non-negotiable acceptance criteria

1. RMB ground move executes immediately, no hold-toggle switch, never halted by Smart Guard.
2. RMB-drag orientation/width and explicit rotate execute normally.
3. RMB attack-unit preserves player intent. A manually requested attack may require approaching to fire; the script should not silently cancel that move.
4. Explicit withdraw, flee/rout, skirmish move and shift-queued movement must not be blocked.
5. During ranged auto-fire/guard, LOS obstruction and out-of-range target changes must NOT create undesired forward chase.
6. Local facing/weapon pivot and shots remain available; non-moving units should not be periodically re-commanded.
7. A mistaken automatic movement stop is worse than a missed correction: **fail open** and prefer player input.

## Why DB `Can't Move` is rejected

Snek's *Hold Position* uses `Can't Move` and 100% movement-speed penalty. Although it prevents auto-reposition, it also blocks manually requested movement until the player disables the ability. Additionally reports of Alt+RMB attack-ground not working while held. This fails criteria 1 and 2, so do not create the native-lock build previously planned:
https://steamcommunity.com/sharedfiles/filedetails/?id=2869716515

## Verified interfaces & limitations

- `battle:register_command_handler` / `bm:register_command_handler_callback`: documented events `Move`, `Move Orientation Width`, `Move Rotation Angle`, `Attack Unit`, `Withdraw`, `Halt`.
- **Major limitation:** `Move` gives only a command name and run flag, not an issuing-unit ID. The event may be fired by a player OR script, so it is NOT a per-unit, player-authenticated signal by itself.
- `Attack Unit` provides a **target unit** via `event:get_unit()`, not a guaranteed subject/selected-unit list.
- `battle:register_unit_selection_handler` can track selection, but selection is circumstantial, not definitive proof that a command was sent to a specific unit.
- `unit:ordered_position()` returns current position during an attack, so do NOT use this as a universal chase detector. It may help only when distinguishing a genuine, separately issued **Move** order.
- `unit:current_target()` / `is_moving()` and `position()` do not reveal the source of movement.
- `unitcontroller:halt()` changes unit orders; `release_control()` does not restore the original attack.
- Documented `register_input_handler` event list concerns camera movement/rotation, NOT a guaranteed RMB click hook.
References:
https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle.html
https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_unit.html
https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_unitcontroller.html

## Track A — conservative Lua v2.2 (possible incremental improvement, NOT a complete fix)

1. Start with **observe-only**, not `halt()`: log command events, selected-unit IDs, current target, center/ordered position, is_moving, and guard/fire-at-will state on a monotonic battle-time timeline.
2. Create explicit per-unit states: `IDLE`, `PLAYER_MOVING`, `ATTACKING`, `SUSPECT_AUTO_REPOSITION`. Do not infer player intent solely from center displacement, an existing target, or attack event.
3. On any `Move` or `Move Orientation Width` command, release pending anchors for tracked selected units and **suppress future halts** for those units until explicit re-arm after the movement completes. Because command events lack affected-unit IDs and source, consider a short conservative GLOBAL suppress window. Never mutate an active Move order.
4. On `Withdraw` likewise suspend the patch (potentially globally if affected units unknown). **Do not cancel Shift MOVE or attack-ground**.
5. On `Attack Unit`, honor a new explicit attack request; it may legitimately require moving into range. Anchor/re-arm only after stable firing state, not simply on an attack event.
6. When target becomes obstructed, only consider intervention when there is strong evidence of an automatically produced reposition rather than a deliberate command, and otherwise **do nothing**. Use hysteresis and rate limit; don't issue recurring `halt()` after a single manual move.
7. Treat `uc:halt()` as last resort and accept that it can cancel the attack; do not claim the solution is smooth without gameplay data.
8. For artillery/manual attack-ground, do not auto-retarget: leave player targets completely alone unless an authoritative order-origin signal exists.
9. Because the command handler itself may trigger on script orders, record the scope of own orders and do not mistake them for the player. This is a guard against false attribution, NOT a reliable player discriminator.

The community *Advanced Auto-Aiming* mod applies an idle-only, fire-at-will+guard gate and immediate script-control release; its author ALSO reports ground-attack override and community notes drifting by about two unit-widths. Treat as a reference and failure case, not proof of a clean fix:
https://steamcommunity.com/sharedfiles/filedetails/?id=3626218075

## Track B — native command/AI-origin conditional (preferred final solution; requires reverse engineering)

Trace and distinguish:
```text
Player RMB -> command-dispatch (Move / MoveAngleWidth / Attack / Withdraw) -> native unit order
                                             |
Ranged auto-fire target selection -> LOS/arc/range evaluation -> *auto reposition request*
                                             |
                             locomotion / formation planning
```
Aim to suppress ONLY ranged-AI-generated **reposition requests while attempting a shot** (optionally guard-mode gated) without changing a unit's generic `can_move` property or canceling normal movement orders.

Required discovery, no address claimed:
- identify the engine-level order type and provenance (player, script, ranged-fire AI);
- identify candidate reposition producer just before movement planner;
- identify safe return/no-reposition handling that leaves fire/rotation and other orders intact;
- guard against engine patch/version drift and multiplayer/replay sensitivity.

Do not hook the universal movement execution function and disable all movement. Do not generate speculative binary patch offsets.

## Offline-first validation (do not burden player with endless ad-hoc tests)

- Static check current game API and executable build, if provided; create a trace-only plugin or script only after the command-provenance signal is understood.
- Isolated trace scenario: (A) RMB ground move, (B) RMB orientation drag, (C) RMB attack within range, (D) RMB out-of-range attack, (E) pure automatic ranged LOS chase, (F) Shift queue, (G) withdraw, (H) ground attack.
- Log unit-ID/order source/target, order timestamps, AI-vs-player provenance if possible, and whether movement was suppressed.
- Tests PASS only if A-D and F-H remain seamless while E auto chase is prevented and firing/pivot preserved.
- No repeat player test requests until a candidate build passes static safety guards and offline replay simulation.

## Immediate project decision

- **Keep v2.1 on main and existing v2.1 release**, no v2.2 build/push.
- **Discard** the toggle-based DB lock as primary route.
- Next work: source-level command-handler study and engine order-provenance discovery, with fail-open Lua only as temporary fallback.
