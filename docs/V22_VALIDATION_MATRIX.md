# Smart Guard v2.2 — validation matrix

Each test must have a control run (vanilla or v2.1 as indicated) and a v2.2 DB prototype run with the SAME battle, unit, target and terrain.

| Case | Test | Expected with Hold ON |
| --- | --- | --- |
| T01 | In-range enemy, clear LOS, front-facing | Fires from original position; no marching |
| T02 | In-range enemy, outside firing arc | Individuals/guns pivot if supported, no whole-unit walk |
| T03 | In-range enemy behind terrain | Does not march to find line of sight; no guarantee of firing |
| T04 | Target exits weapon range | No chase; range limits remain |
| T05 | Target dies or user retargets | No uncommanded position drift |
| T06 | Manual move while Hold ON | A blocked move is expected; toggling OFF restores movement |
| T07 | Guard ON/OFF, Fire at Will ON/OFF | Confirm no unexpected order override or mode conflict |
| T08 | Ranged infantry, artillery, weapon team | Test separately; artillery attack-ground and retargeting |
| T09 | Skirmish Mode, melee threat, routing | Never trap units in unintended permanent state without recovery |
| T10 | Current-game vanilla unit roster | No invalid foreign keys or database startup crash |
| T11 | Campaign, custom battle, with overhaul mod | Only claim tested modes; confirm installation/removal/save behavior |
| T12 | L10+ player units, 20-minute battle | No repeated halt spam, path jitter, unexpected performance loss |
| T13 | Release toggle -> move -> re-enable -> shoot | No stale movement lock, no unusable ability |
| T14 | Other mod controlling unit | No accidental Lua reorders; check DB collisions |

Recommended telemetry: unit center position every 100-200ms for **observation only**, count of manual orders, shots fired/ammo delta, UI ability state, and timeline of target changes. No `halt()` in the initial v2.2 test pack.

Pass gate: T01-T06 and T10/T13 mandatory; document any exceptions for artillery and mods before a public release. Engine testing on game build 9.x is not yet performed.
