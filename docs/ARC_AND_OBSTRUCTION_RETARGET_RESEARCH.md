# Smart Guard: firing arc and obstructed-target alternatives (2026-11-02)

Status: research only; NO new pack, game test or release. Previous Don't Move DB toggle remains rejected because RMB must work immediately.

## 1. DB facing instead of movement lock

A 2025-08-10 Warhammer III mod author reports significant reduction of autonomous ranged formation shuffle by increasing battle-entity fire_arc to 180 degrees, setting defense_line_pivot_factor to 0.5, and changing entity_action_attack_matched_attacker_facing_limit_angle_deg / defender to 360. These are reported results, NOT locally reproduced, and not guaranteed on current 9.x.

Source: https://www.bilibili.com/video/BV126tezcEbw/

Separate up-to-date DB schema guide names battle_entities_tables.fire_arc_close, turn_speed and allow_static_turn_anim / allow_turn_to_move_anim. Do NOT equate fire_arc and fire_arc_close until the current RPFM schema is read. Do not change animation flags or broad global melee-facing keys as first experiment.

Source: https://tw-modding-cn.com/warhammer3/rpfm-%E8%A1%A8%E5%8D%95%E8%AF%A6%E8%A7%A3/%E9%99%86%E6%88%98%E5%8D%95%E4%BD%8D%E8%AF%A6%E8%A7%A3/

Isolated prototype: select 1 archer and 1 direct-fire infantry unit; validate schema and effects of 90/120/140/180 candidates one at a time; quantify turning and center displacement. Shared entity keys also affect AI, other vanilla units and mods. Wider arcs can alter balance, shooting animations and friendly fire. The tactic targets turning/re-form motion, not LOS-driven chasing.

## 2. Retargeting when obstructed

Existing Advanced Auto-Aiming mod selects targets when Guard + Fire at Will and unit idle, then issues attack and releases control. Author warns this can override artillery attack-ground, and clarified in comments that native obstruction is NOT resolved merely by target scoring. Users reported drifting several model widths.

Source: https://steamcommunity.com/sharedfiles/filedetails/?id=3626218075

Documented battle Lua: unit:current_target(), unit:is_behaviour_active('defend'), unit:is_behaviour_active('fire_at_will'), unit:unit_in_range(candidate), unit:is_valid_target(), unit:position(), unit:bearing(), unit:ammo_left(), unit:is_moving(), and unitcontroller:attack_unit(target). This attack instruction MAY itself trigger movement to acquire fire; the switch alone is not a perfect stop.

Unit visibility-to-alliance is fog-of-war/terrain occlusion at the army-alliance level, NOT a reliable shooter-target ballistic LOS test.

Sources: https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_unit.html and https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_unitcontroller.html

## 3. NEW read-only investigation: CCO status and firing

WH3 CCO documentation exposes CcoBattleUnit.StatusList, StatusMostImportant, IsFiringMissiles, ReloadPercentMax and PrimaryAmmoPercent. CcoUiUnitStatusRecord exposes Key, IconPath, Tooltip. A documented example in community code reads CcoBattleUnit via common.get_context_value keyed by unit:unique_ui_id().

References: https://chadvandy.github.io/tw_modding_resources/WH3/cco/documentation.html ; https://github.com/wustzhh/Warhammer3Mod/blob/main/docs/%E6%88%98%E9%94%A43%E5%8E%9F%E7%89%88%E6%88%98%E6%96%97%E8%84%9A%E6%9C%AC%E5%87%BD%E6%95%B0%E6%95%B4.md

UNKNOWN (must not guess): whether UI's in-game yellow 'Obstructed' firing icon maps to a stable CcoUiUnitStatusRecord.Key, whether this list is accessible in battle Lua by UI ID, and whether that status really distinguishes obstruction from reload and range. First build OBSERVE-ONLY logging; no scripted attack/halt.

## 4. Fail-open algorithm IF reliable obstruction state is found

1. Filter player ranged units, ammunition > 0, Guard + Fire-at-Will on, standing still and not in melee/routing.
2. Preserve all RMB Move/drag/attack, Shift queues, Withdraw and artillery manual attack-ground; command handler only provides limited provenance and can be script-induced, so refrain from retarget when intent is uncertain.
3. Detect obstruction by verified CCO status. Ammo unchanged or IsFiringMissiles=false alone is insufficient (long reload, idle, ammo special rules).
4. Search enemy units in range / approximately in front / valid & visible; this is ONLY candidate scoring, not a guarantee of LOS.
5. After persistent obstruction, switch to materially better candidate, cool down for multiple seconds, and blacklist unsuccessful candidates temporarily to prevent ping-pong.
6. If no valid candidate: do NOTHING; never automatically halt an explicit player command. Obstruction-driven movement may still occur in this case, requiring native AI investigation.
7. Do not touch artillery until explicit ground-target recognition is proven.

## 5. Implementation order

A. Validate narrowly scoped firing-arc DB change in custom battle (turning, actual firing, RMB unaffected).
B. Add read-only CCO observation to find exact obstructed status key, if it exists, and correlate with ammo and motion.
C. Only then trial conditional retarget in an isolated prototype, fail-open when no high-confidence user-intent signal.
D. Native ranged-AI reposition producer remains long-term path for perfect LOS suppression without movement lock.

Current v2.1 public Release/main remain untouched.