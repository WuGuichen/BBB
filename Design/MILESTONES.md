# MILESTONES — 制作计划

> 版本 0.2 | 2026-05-27
>
> 本文描述开发阶段、验收条件和风险。技术映射见 [TECHNICAL.md](TECHNICAL.md)，玩法设计见 [GDD.md](GDD.md)。

---

# 1. 阶段总览

| 阶段 | 目标 | 交付物 | 依赖 |
|------|------|--------|------|
| **M0** | 项目骨架 | VVV 游戏层 + Composition Root + Debug HUD | 无 |
| **M1** | 确定性移动 | AABB 房间 + 单体移动 + 避障 + Replay hash | M0 |
| **M2** | AI 闭环 + 任务 | AI Planner 驱动行为 + 拾取任务 + 战后报告 | M1 + ⚠️ P0 AI 扩展 |
| **M3** | 基础战斗 | 敌人 + Bite/Charge + 伤害/死亡 + 战斗日志 | M2 |
| **M4** | 身体部件 v0 | 部件→属性/Action/Fact 映射 + 构筑影响任务结果 | M3 |
| **M5** | AI Profile v0 | 5 个预制性格 + 行为差异可感知 | M4 + ⚠️ P0 AI 扩展 |
| **M6** | 生态地图 v0 | 地图标签 + 物种分布 + 多房间 | M3 + ⚠️ P1 NavGraph |
| **M7** | 玩家间接指挥 | 信标/信号 → AI Fact 变化 | M5 |
| **M8** | 战后分析产品化 | 失败分类 + 改进建议 + 批量仿真 | M2 |

**M2 和 M5 依赖 P0 框架 AI 扩展**（见 [TECHNICAL.md §3](TECHNICAL.md#3-框架缺口分析)）。框架调整和游戏层开发应并行推进。

---

# 2. 各阶段详情

## M0：项目收敛与切片骨架

**目标**：建立 VVV 游戏层，不再只做框架 showcase。

要做：
1. 新建 `Assets/Scripts/VVV/` 游戏层目录
2. 新建 `VVVRuntimeCompositionRoot`
3. 接入 `RuntimeHost` + `RuntimeCommandBuffer` + `CombatPhysicsWorld`
4. 建立最小 Debug HUD（UI Toolkit）

验收：
- Play 后出现场景、HUD、1 个占位生物
- RuntimeHost 正常 Tick
- Reset 后状态一致

不做：美术、AI、敌人、战斗

---

## M1：确定性地图 + 单体移动

**目标**：一只生物能在 AABB 房间里移动、避障、到达目标。

复用：`CombatPhysicsWorld` + `CombatKinematicMotor`

要做：
1. `EcoMapDefinition` + `EcoRoomDefinition`
2. AABB 障碍生成到 CombatPhysicsWorld
3. `CreatureMotionAdapter`（包一层，不重写运动管线）
4. `MoveToTargetCommand`
5. 视觉同步（占位几何体）

验收：
- 生物从起点走到资源点，撞墙不穿透
- 重复运行 Replay hash 一致
- 不同帧率不改变结果

---

## M2：AI Planner + 拾取任务闭环 ⚠️ 依赖 P0

**目标**：生物能根据事实和目标自主选择行动，完成回收任务。

复用：`Runtime AI Planner` + `GameplayComponentWorld`

要做：
1. `CreatureFactKeys`（resource.visible / exit.visible / inventory.full / hp.low / enemy.near）
2. `CreatureAiWorldStateProjector`
3. 基础 Goal（collect.resource / return.exit / avoid.death）和 Action（MoveTo / PickUp / Return / Flee / Wait）
4. `MissionRuntimeModule`（CollectResource + ReturnToExit 目标）
5. `MissionReport`（成功/失败、耗时、触发规则、失败原因）

验收：
- 默认 AI（TaskFirst）可完成简单拾取任务
- 切换为 Hunter 后生物更容易追敌偏离目标
- 切换为 Coward 后生物更容易撤退
- 战后报告能指出失败原因

**⚠️ 阻塞项**：需要框架补 AI Planner Trace（缺口 C），否则无法生成报告。

---

## M3：基础战斗

**目标**：敌人出现后，生物能战斗、受伤、死亡或撤退。

复用：`CombatPhysicsWorld.Query` + `Gameplay Attributes/Buffs`

要做：
1. `CreatureCombatRuntimeModule`
2. Bite（Sphere query + DamageByAttackDefense）
3. Charge（Capsule sweep + 击退）
4. 敌人 AI（巡逻 + 看到玩家后攻击）
5. 死亡 → 尸体状态
6. 战斗事件日志

验收：
- 玩家生物能咬死弱敌，强敌能杀死玩家生物
- 战报说明死亡原因
- Replay 结果一致

不做：连招、打断、部位破坏

---

## M4：身体部件 v0

**目标**：玩家可通过身体部件影响任务结果。

复用：`AttributeStore.AddModifier` + `ConfigModifierFactory`

要做：
1. `BodyPartDefinitionConfig`（Core x2 / Movement x2 / Sensor x2 / Weapon x2 / Defense x2 / Brain x2）
2. 部件→属性修改器映射
3. 部件→Action 解锁映射
4. 部件→Fact 解锁映射
5. 构筑验证报告

验收：
- fast_legs 构筑更快完成回收任务
- armor_shell 构筑更容易活过战斗
- smell_sensor 构筑能在无视野时找到资源

性价比简化：模型暂不变，只改颜色/图标/HUD 文案。

---

## M5：AI Profile v0 ⚠️ 依赖 P0

**目标**：玩家可选不同 AI 性格，行为差异可感知。

要做：
1. `AiProfileDefinition`（Scavenger / Hunter / Coward / Guardian / TaskFirst）
2. Goal priority modifier（⚠️ 需框架补缺口 A）
3. Action cost modifier（⚠️ 需框架补缺口 B）
4. Brain Debug View（显示当前 goal/action/facts/triggered rules）

验收：
- 同一身体换不同 Profile 后任务行为明显不同
- Brain View 能解释差异原因

---

## M6：生态地图 v0

**目标**：地图带语义标签，物种分布有逻辑。

复用：`CombatPhysicsWorld`（碰撞体）

要做：
1. `HabitatZoneDefinition`（light/terrain/hazard/resource 标签）
2. `SpeciesSpawnProfile` + SpawnScore 计算
3. 多房间地图（废弃温室 / 酸性洞穴 / 机械废墟）
4. 第一版野生物种（prey_bug / scavenger_bug / guard_beast）

验收：
- 不同标签的地图 spawn 不同物种
- 酸池地图有酸抗需求

**⚠️ 可选依赖**：HazardVolume（缺口 E+F）和 NavGraph（缺口 H）。如果框架未补，游戏层自建最小实现。

---

## M7：玩家间接指挥

**目标**：玩家能影响 AI，但不能直接操控。

要做：
1. `PlayerSignalCommand`（Move Beacon / Recall / Threat Mark / Resource Mark）
2. 信号写入 AiWorldState facts
3. 不同 Profile 对信号的不同响应

验收：
- TaskFirst 响应 Recall，Hunter 响应 Threat Mark
- Coward 把 Threat Mark 当作避开区域

---

## M8：战后分析产品化

**目标**：每次失败都有明确原因和可执行建议。

要做：
1. `FailureClassifier`（died.to.enemy / died.to.hazard / ignored.objective / overaggressive / low.sensor）
2. `BuildRecommendationEngine`（硬编码规则 → 建议修改部件/AI）
3. 批量仿真（100 次不同 Profile 对比成功率/死亡率/失败分类分布）

验收：
- 每次失败有明确原因
- 报告指出至少一个可改构筑点
- 批量仿真输出可指导策划平衡

---

# 3. 最小竖切内容清单

第一版真的只需要这些：

| 类别 | 内容 |
|------|------|
| **地图** | Lab Room 01：1 封闭房间 + 3 AABB 障碍 + 1 资源点 + 1 出口 + 1 敌人出生点 |
| **生物** | Player Creature：HP / MoveSpeed / Attack / Defense / Inventory |
| **敌人** | Dummy Beast：巡逻 + 看到玩家后攻击 |
| **AI Profile** | TaskFirst / Hunter / Coward |
| **动作** | MoveToResource / PickUpResource / MoveToExit / BiteEnemy / FleeFromThreat / Wait |
| **UI** | Start / Pause / Step / Reset / Save / Load / Brain View / Mission Report |
| **报告** | Success/Failure / Final HP / Resource count / Selected actions / Failure reason / Suggested change |

---

# 4. 风险清单

| 风险 | 应对 |
|------|------|
| AI 看起来像乱跑 | M2 就做 Brain View，每个 action 有 reason |
| 系统太复杂没人能调 | 先只给 3 个 AI Profile，不开放自由编辑 |
| 战斗不好玩 | 第一版战斗只验证 AI 策略（打/跑/绕/完成任务），不追求手感 |
| 美术拖住开发 | 全用占位几何体，身体部件只在 HUD 展示 |
| 框架 AI 扩展未到位 | M2/M5 可先用硬编码优先级做 prototype，等框架补能力后切换 |

---

# 5. 暂缓清单

明确不做：

| 功能 | 原因 |
|------|------|
| 程序化身体建模 | 玩法未验证，先用数值+占位体 |
| 复杂 IK 步态 | 同上 |
| 玩家 AI 图编辑器 | 先用预制 Profile，后期再开放 |
| 繁殖/遗传 | 太复杂，不在 MVP |
| 多人蓝图对战 | 需要网络层，不在 MVP |
| 完整 Mod SDK | 需要稳定 API，不在 MVP |
| 动态天气/生态演化 | 先做静态生态标签 |
| 正式美术 | 全用占位体 |
| CharacterAction 集成 | 代码完成但未接入管线，等基础闭环成立后再评估 |

---

# 6. 框架调整 Issue 建议（暂不创建）

待确认后在 WGameFramework 仓库创建：

| 优先级 | Issue | 内容 |
|--------|-------|------|
| P0 | [AI] Goal Priority Modifier | 给 PriorityGoalSelector 增加 IGoalPriorityModifier 接口 |
| P0 | [AI] Action Cost Modifier | 给 IAiAction 增加 cost modifier 机制 |
| P0 | [AI] PlannerTrace Output | 给 SequentialPlanner 增加 trace 输出 + DebugUI 集成 |
| P1 | [Combat] HazardVolume | 新增 HazardVolume 概念 + tick 伤害/Buff 系统 |
| P1 | [Navigation] Minimal NavGraph | 纯 C# A* + Waypoint/Edge + portal 限制 |
| P2 | [Gameplay] Inventory Component | 通用 Inventory 组件 + hash + SaveState |
