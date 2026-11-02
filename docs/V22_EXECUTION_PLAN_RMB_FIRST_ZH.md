# Smart Guard v2.2 — RMB-first 实证研究与实施计划
日期：2026-11-02。**研究方案，未制作新 pack，未经过实机验证。** 本文是当前研究的执行入口；旧的 Can't Move 方案已弃用，v2.1 Release 保持不变。

## 目标与不得破坏的行为

**尽可能阻止远程单位因为自动射击而自行寻找射击位置，而不是限制单位通用移动能力。**

必须完整保留：(1) RMB 地面移动、拖拽变阵、旋转，(2) Shift MOVE 队列，(3) 玩家 RMB 指定敌军攻击（允许走近射程），(4) 撤退和散兵行为，(5) 火炮攻击地面和明确指定的目标。严禁常驻 Can't Move / 100% 减速，严禁不断调用 halt() 纠偏。用户操作冲突视为 **立即否决**，误判时应不执行脚本命令。

## 源码已知缺陷（v2.1）

原文件 `script/battle/mod/smart_guard.lua` 277 行：攻击目标超过 `missile_range()` 时在第 146–149 行立即 `halt_and_release`；100ms 定时器在第 194–239 行检测单位中心偏移 >1.2m 就可能再次 `halt_and_release`。单位中心移动不代表 AI 发出的寻位命令，且 halt 可能取消玩家攻击命令。

**新实验一定不能与 v2.1 共同加载**，否则观察数据会被原拦截器污染。

## 研究 A — P0 离线核对（首步）

1. 复核 CA Battle API 的 `unique_ui_id`, `position`, `bearing`, `ordered_position`, `is_moving`, `current_target`, `unit_in_range`, `ammo_left`, `is_behaviour_active`；特别留意 `ordered_position` 在攻击时返回当前位置，不能直接识别追击。
2. 核对 `register_command_handler`：`Move`, `Move Orientation Width`, `Move Rotation Angle`, `Attack Unit`, `Withdraw`。**重要：`Move` 不提供确定的发令单位 ID / 玩家来源；`Attack Unit` 的 `event:get_unit()` 是被攻击目标，不是己方射手。** Selection 只能提供弱关联，不能硬判。
3. 使用当前 9.x 游戏数据和 RPFM Schema 追踪 `land_units` → `battle_entities`、`missile_weapons`、`projectiles`。查清 `fire_arc` 和 `fire_arc_close` 到底在哪张表、是否同义、单位是什么及实际默认值，查出共享实体的单位数量。未核对前不对全体填 180° 或 360°。

交付：`api_capabilities.csv`、`db_parameter_inventory.csv`，每一项都标明“已确认 / 未确认 / 不可用”与证据。

## 研究 B — P1 只读的 CCO 遮挡探针（第一优先级）

已有公开 CCO 文档列出：
- `CcoBattleUnit.StatusList`, `StatusMostImportant`
- `CcoBattleUnit.IsFiringMissiles`, `ReloadPercentMax`, `PrimaryAmmoPercent`
- `CcoUiUnitStatusRecord.Key`, `Tooltip`, `IconPath`

**但文档存在≠战斗 Lua 能读取≠黄色射击受阻图标必有独立 Key。** 首先试 `common.get_context_value("CcoBattleUnit", tostring(uid), "IsFiringMissiles")`，对标量访问的成功/失败输出类型。之后尝试 `StatusList.Size`，并逐一验证正确的列表元素表达式，不预设 `.At(i).Key` 一定成立；失败后停止重复调用并记明错误。

构建单独的 `SG_OBSERVE_ONLY` Lua 研究包：
- 只监控 2–4 个指定单位，200ms 采样；命令/状态变化即时记一条。
- 记录：`time_ms, scenario, game_build, unit_uid, target_uid, x,z,bearing,moving,guard,fire_at_will,ammo,cco_status_keys,is_firing,reload_percent,command_name,selection_ids,read_errors`。
- 原始 `nil`、`false` 和 API 错误必须区分，日志不可使用全静默 `pcall`。
- 只读，不调用 `halt`、`attack_unit`、`goto`、`release_control`，不改实际玩家单位行为。

最少测试情形：
| ID | 场景 | 区分信号 |
|---|---|---|
| S1 | 正面、无遮挡、射程内开火 | 正常基线 |
| S2 | 正在装填 | 不射 != 遮挡 |
| S3 | 友军挡射线 | 遮挡候选 |
| S4 | 地形/建筑挡射线 | 遮挡候选 |
| S5 | 目标出射程 | 射程 != 遮挡 |
| S6 | 目标在侧后方 | 射界/转向 |
| S7 | RMB 移动、拖拽、明确 RMB 攻击 | 指令保护 |
| S8 | Shift 队列、Withdraw、火炮攻击地面 | 指令保护 |

产物：`sg_observe.jsonl`、`cco_expression_probe.csv`、`status_mapping.csv` 以及时间线、误判分析。**继续门槛**：必须证明状态可读且能稳定区分 S2/S3/S4/S5。若没有稳定受阻状态，自动转火降为非默认实验，不能只用 `is_firing=false` 就下令。

## 研究 C — P2 DB 单变量转向实验（与 P1 静态并行）

此路线只解决“转向/重新列阵导致移动”，并不天然解决 LOS 遮挡寻路。
1. 在 Schema 已确认后，选 1 支弓手和 1 支直射枪兵，按共享 `battle_entity` Key 确定实验影响范围。
2. 原版 A 与修改 B：**一次只改一个字段**；先射界，再阵型 pivot，最后才考虑全局攻击朝向 KV。不得第一轮同时大改 180/360。
3. 场景：正前方/左右侧/后方敌人、同方向切换目标、RMB 拖拽、移动后原地射击、地形遮挡。
4. 每个 A/B 场景记录中心位移最大值和 P95、是否开火、开火延迟、模型转向、友伤/背射、玩家 RMB 指令是否生效。每组有重复记录，避免仅靠主观观感。
5. 如字段共享会影响大量单位/AI，实验 pack 不进入 Release；必要时克隆实体，但必须逐条检查有效引用和动画。

产物：最小 DB 对照 pack + `db_diff.csv` + 比较表。与“智能转火”代码完全分离。

## 研究 D — P3 条件式自动转火（只在 P1 通过后实现）

基本算法：
```text
if RMB/Shift/撤退/玩家指定攻击可能正在执行: 不处理
if 非玩家受控、非有弹药远程、未开 Guard+Fire at Will: 不处理
if 移动中、近战中、溃逃、火炮手动攻击地面: 不处理
if 不能高置信区分受阻和装填: 不处理
if 受阻信号未持续达到观测确定的窗口: 不处理
候选 = 射程内 + 有效 + 正面射界内 + 非近期失败目标
if 备选目标明显优于当前目标 且非玩家手动锁定目标:
    仅一次 attack_unit(备选); release_control()
    进入冷却并观测是否真的开火；失败则暂时拉黑该目标
else:
    不下任何命令（不 halt）
```

**注意：`unit_in_range()` 只能判射程，不能证明射线通畅；`attack_unit()` 自身可能触发 AI 走位。** 候选排序只是一种尝试，不得宣称全局保证“不会乱动”。要有转火滞回、冷却、失败黑名单，绝不每 tick 改目标。

玩家意图识别是致命门槛：Lua `Move` 事件没有可靠逐单位来源，不能靠“当前选中单位”就认为一定是目标单位。如果不能证明手动攻击不被覆盖，默认只在无手动指定目标时开自动转火，或者停止这部分开发。

产物：P1 数据驱动的 Lua 状态机源码、离线模拟用事件记录、隔离测试 pack；**任何一次覆盖玩家 RMB/Shift/Withdraw/attack-ground 都否决**。

## 研究 E — P4 Native（必要时）

如果 P2 有限、P3 也无法阻止没有替代目标时的自动寻路，才进入原生分析：区分玩家 RMB order 与 ranged AI LOS/射界引发的 reposition request，只在射击寻位任务生成时介入。绝不拦截通用单位移动执行函数。不凭空给出汇编地址；需匹配游戏 build、调用链与可回滚证据。此方向与 BSC 的 Shift MOVE 修复严格分开。

## 开发顺序与停止条件

| 阶段 | 工作性质 | 输出 | 通过条件 |
|---|---|---|---|
| P0 | 纯静态 | API/DB 清单 | 关键接口与字段有实际定义 |
| P1 | 只读探针 | 一次有计划的采集 | 射击受阻与装填/出射程可区分 |
| P2 | DB A/B | 最小参数试验 | RMB 不变且转向挪位明显下降 |
| P3 | 条件转火 | 离线仿真+少量实战 | 手动命令干扰为 0，正常射击保留 |
| P4 | 原生分析（按需） | 原生命令来源证据 | 只拦截 AI-origin 自动寻位 |

尽量只安排**两次有目的的玩家实机测试**：第一次 P1 只读日志；第二次有候选结果后对照 A/B。中间的分析与模拟由开发侧完成。游戏安装、9.x RPFM Schema 与 `db.pack` 如果不可直接访问，不应臆造 CA 字段和编译 pack。

## 验收指标

- 玩家 RMB/Shift/拖拽/撤退/攻击地面命令冲突：**0 次容忍**。
- 能射击时仍要开火，记录射击时间、弹药变化；不能为了静止而变成不攻击。
- 没有玩家移动命令时，单位中心的最大位移与 P95 相对原版显著下降（改善阈值在基线测完后确定，不继续照搬 0.4/1.2m）。
- 无反复转火、失败目标循环、异常日志膨胀或稳定性损失。
- 火炮、抛射/直射、骑射须分别测，不从弓手推断全单位适用。

## 参考

- CA battle: https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle.html
- CA unit: https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_unit.html
- CA CCO: https://chadvandy.github.io/tw_modding_resources/WH3/cco/documentation.html
- CA controller: https://chadvandy.github.io/tw_modding_resources/WH3/battle/battle_unitcontroller.html
- RPFM 寻找原版数据: https://tw-modding-cn.com/warhammer3/rpfm/%E6%9F%A5%E6%89%BE%E5%8E%9F%E7%89%88%E6%95%B0%E6%8D%AE/
- Hold Position (不符合 RMB 要求): https://steamcommunity.com/sharedfiles/filedetails/?id=2869716515
- Advanced Auto-Aiming (自动选目标参考): https://steamcommunity.com/sharedfiles/filedetails/?id=3626218075
