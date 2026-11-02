# Smart Guard v2.1 — Total War: WARHAMMER III

**Smart Guard** (also described as **Ranged – Hold Your Position**) is a battle Lua mod intended to reduce unwanted movement of ranged units while they attack with Guard Mode enabled.

It monitors a ranged unit's movement after an attack command and can issue a halt if the unit drifts too far from its position. This is a Lua-level workaround, **not** a patch to the game's underlying movement AI.

## Download

Go to [Releases](https://github.com/Twilightsp123/zzz-smart-guard/releases) and download `zzz_smart_guard_v2.1.zip`.

The archive contains **`zzz_smart_guard.pack`**, the mod file. No external installer is included.

## Installation

1. Download the release ZIP and extract `zzz_smart_guard.pack`.
2. Copy the `.pack` file into the **data** directory inside your *Total War: WARHAMMER III* installation.
3. Enable the mod using a mod manager / launcher that supports your installation. For Epic Games installations, find the game's installed directory via the Epic launcher first.
4. Start a battle and test a ranged unit with **Guard Mode** enabled.

**Epic Games compatibility has not been verified in-game.** Availability of mod activation depends on the game version, launcher and mod manager; simply copying a file might not be sufficient on every setup.

## Important notes

- Intended for ranged units using Guard Mode; it does **not** enable withdrawal for all units.
- Does not change the native executable, underlying pathfinding, or individual-soldier movement.
- May conflict with other mods that issue movement/attack orders.
- This is the original supplied **v2.1** package, published without modifying its `.pack` contents.
- Diagnostics are disabled in the supplied Lua script (`ENABLE_LOGGING = false`).

## Integrity

SHA-256 of the **`zzz_smart_guard.pack`** inside the release ZIP:

`f3f9a5cd4d5afe93dfa35f3f971018d4bbdd511b434d2b081d67ee7518e77010`

SHA-256 of the **`zzz_smart_guard_v2.1.zip`** (original supplied archive):

`71ba80732e7d3c422fcf89d30a2c1754110469e207861dc4f4db24ea5eb407df`

## License

See the repository's existing [LICENSE](LICENSE).