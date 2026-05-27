# MILESTONES — 制作计划

> 版本 0.3 | 2026-05-27
>
> 本文描述开发阶段、验收条件和风险。技术映射见 [TECHNICAL.md](TECHNICAL.md)，玩法设计见 [GDD.md](GDD.md)。

---

# 1. 阶段总览

| 阶段 | 目标 | 核心验证 |
|------|------|----------|
| **M0** | 游戏层骨架 | RuntimeHost 能 Tick，占位生物可见 |
| **M1** | 确定性移动 | 生物在 AABB 房间中移动不穿墙，Replay hash 一致 |
| **M2** | 单一 AI 拾取任务 | TaskFirst 完成回收任务，失败时记录决策 |
| **M3** | Brain View + Mission Report v0 | 玩家能看懂 AI 为什么这么做 |
| **M4** | 基础战斗 | 敌人给任务制造压力，死亡原因进入报告 |
| **M5** | 身体部件 v0 | 身体配置影响任务结果 |
| **M6** | AI Profile v0 | 同一身体不同性格行为明显不同 |
| **M7** | 生态地图 v0 | 地图标签影响物种和构筑选择 |
| **M8** | 玩家间接指挥 | 信标/信号影响 AI 行为 |
| **M9** | 批量仿真 | 自动跑 100 次，输出成功率和失败类型 |

**关键原则**：M2 不要求 Hunter/Coward，只用 TaskFirst。AI Profile 在 M6 才正式做。Brain View 和 Mission Report 在 M3 就做最小版，不等到 M8。

---

# 2. 各阶段详情

## M0：游戏层骨架

**目标**：VVV 游戏层成立，RuntimeHost 正常 Tick。

要做：
1. 新建 `Assets/Scripts/VVV/` 游戏层目录
2. 新建 `VVVRuntimeCompositionRoot`
3. 接入 `RuntimeHost` + `RuntimeCommandBuffer`
4. 建立最小 Debug HUD（UI Toolkit）

验收：
- Play 后有占位生物
- Frame 增长
- Reset 后状态一致

不做：美术、AI、敌人、战斗、碰撞

---

## M1：确定性移动

**目标**：单体生物在 AABB 房间中移动到目标点。

复用：`CombatPhysicsWorld` + `CombatKinematicMotor`

要做：
1. `EcoMapDefinition` + `EcoRoomDefinition`
2. AABB 障碍生成到 CombatPhysicsWorld
3. `CreatureMotionAdapter`（包一层，不重写运动管线）
4. `MoveToTargetCommand`
5. 视觉同步（占位几何体）

验收：
- 不穿墙
- 能到达指定点
- Replay hash 一致
- 不同帧率结果一致

---

## M2：单一 AI 拾取任务

**目标**：TaskFirst AI 完成资源拾取并返回出口。

**只做一个 AI（TaskFirst），不要求 Hunter/Coward。**

复用：`Runtime AI Planner` + `GameplayComponentWorld`

要做：
1. `CreatureFactKeys`（resource.visible / exit.visible / inventory.full / hp.low）
2. `CreatureAiWorldStateProjector`
3. 基础 Goal（collect.resource / return.exit / avoid.death）和 Action（MoveTo / PickUp / Return / Flee / Wait）
4. `CreatureAiProfileRuntimeAdapter`（硬编码 TaskFirst 的 goals/actions，见 [TECHNICAL §3.2](TECHNICAL.md#32-核心策略游戏层先硬编码验证)）
5. `CreatureAiDecisionTrace`（游戏层自建 trace，见 [TECHNICAL §3.2](TECHNICAL.md#32-核心策略游戏层先硬编码验证)）
6. `MissionRuntimeModule`（CollectResource + ReturnToExit 目标）

验收：
- TaskFirst 在 LabRoom01 成功率 ≥ 80%
- 失败时记录最后 10 次决策（frame、facts、selected goal/action、reason）
- 同一输入 Replay hash 100% 一致

---

## M3：Brain View + Mission Report v0

**目标**：玩家能看懂 AI 为什么这么做。

要做：
1. Brain Debug HUD（UI Toolkit）
   - 显示当前 facts（resource.visible / hp.low / enemy.near 等）
   - 显示当前 goal + priority
   - 显示当前 action + cost
   - 显示 selected reason
2. Mission Report（任务结束后显示）
   - 成功/失败
   - 耗时（frame 数）
   - 资源收益
   - 失败分类（timeout / died / lost.path / ignored.objective）
   - 最后 10 次 AI 决策时间线

验收：
- Brain View 关键 Label 文本非空，颜色 alpha > 0
- 任务失败后，报告能指出至少一个失败原因
- 玩家能根据报告说出"下次我该怎么改"

---

## M4：基础战斗

**目标**：敌人给任务制造压力。

只做：Bite（Sphere query + DamageByAttackDefense）+ Charge（Capsule sweep + 击退）+ HP + Death + Corpse placeholder

复用：`CombatPhysicsWorld.Query` + `Gameplay Attributes/Buffs`

要做：
1. `CreatureCombatRuntimeModule`
2. Bite + Charge 攻击
3. 敌人 AI（巡逻 + 看到玩家后攻击）
4. 死亡 → 尸体状态
5. 战斗事件进入 Decision Trace

验收：
- 弱敌可被 Bite 击杀
- 强敌可杀死玩家生物
- 死亡原因进入 Mission Report（died.to.enemy）
- Replay 结果一致

不做：连招、打断、部位破坏、Projectile、Slash、群体战斗

---

## M5：身体部件 v0

**目标**：身体配置影响任务结果。

复用：`AttributeStore.AddModifier` + `ConfigModifierFactory`

要做：
1. `BodyPartDefinitionConfig` — 6 类，每类 2 个：
   - Core: light_core / heavy_core
   - Movement: fast_legs / stable_legs
   - Sensor: basic_eye / smell_sensor
   - Weapon: bite_jaw / charge_horn
   - Defense: thin_skin / armor_shell
   - Brain: primitive_brain / tactical_brain
2. 部件→属性修改器映射
3. 部件→Action 解锁映射（charge_horn 解锁 Charge）
4. 部件→Fact 解锁映射（smell_sensor 解锁 corpse.near / resource.smell）

验收：
- fast_legs 平均完成时间比 stable_legs 短 ≥ 15%
- armor_shell 生存率比 thin_skin 高 ≥ 20%
- smell_sensor 能在无视野时找到资源（basic_eye 不能）
- charge_horn 解锁 Charge 攻击

性价比简化：模型暂不变，只改颜色/图标/HUD 文案。

---

## M6：AI Profile v0

**目标**：同一身体，不同性格行为明显不同。

复用：`CreatureAiProfileRuntimeAdapter`（M2 已建，现在扩展到 3 个 Profile）

要做：
1. 扩展 `CreatureAiProfileRuntimeAdapter` 支持 TaskFirst / Hunter / Coward
2. Brain Debug View 显示 Profile 名称和 goal/action 差异

验收：
- Hunter 攻击动作次数比 TaskFirst 高 ≥ 50%
- Coward 在 enemy.near + hp.low 时，Flee 选择率 ≥ 80%
- Brain View 能解释三个 Profile 的行为差异
- 同一 Replay 输入下，三个 Profile 的 hash 不同（因为 plan 不同）

---

## M7：生态地图 v0

**目标**：地图标签影响物种和构筑选择。

复用：`CombatPhysicsWorld`（碰撞体）

分两步：

### M7a：地图标签 + 多房间

要做：
1. `HabitatZoneDefinition`（light/terrain/hazard/resource 标签）
2. 多房间地图（废弃温室 / 酸性洞穴 / 机械废墟）
3. Room + Portal 结构

验收：
- 不同房间有不同的 terrain/hazard 标签
- Portal 正确连接房间

### M7b：物种 SpawnProfile + 行为

要做：
1. `SpeciesSpawnProfile` + SpawnScore 计算
2. prey_bug（小型猎物）+ guard_beast（重甲守卫）
3. 游戏层 GameHazardVolume（酸池伤害）

验收：
- prey_bug 出现在开阔资源区
- guard_beast 守着高价值资源点
- 酸池每 tick 对区域内实体造成伤害

---

## M8：玩家间接指挥

**目标**：玩家能通过信号影响 AI。

要做：
1. `PlayerSignalCommand`（Move Beacon / Recall Beacon）
2. 信号写入 AiWorldState facts
3. 不同 Profile 对信号的不同响应

验收：
- Recall Beacon 能提高 Return 行为选择率 ≥ 30%
- Hunter 对 Recall 的响应弱于 TaskFirst（因为 defeat.enemy 优先级高）
- Coward 对 Recall 的响应强于 Hunter

---

## M9：批量仿真与平衡报告

**目标**：自动跑 100 次任务，输出成功率和失败类型。

复用：`FrameworkSimulationBatchRunner`（框架已有）

要做：
1. 100 次 TaskFirst collect mission
2. 100 次 Hunter collect mission
3. 100 次 Coward collect mission
4. fast_legs vs armor_shell 对比

验收：
- 输出每个 Profile 的成功率
- 输出平均耗时
- 输出死亡率
- 输出失败分类分布（died / timeout / ignored.objective 等）
- 输出 fast_legs vs armor_shell 的完成时间差异

---

# 3. 最小竖切内容清单

第一版真的只需要这些：

| 类别 | 内容 |
|------|------|
| **地图** | Lab Room 01：1 封闭房间 + 3 AABB 障碍 + 1 资源点 + 1 出口 + 1 敌人出生点 |
| **生物** | Player Creature：HP / MoveSpeed / Attack / Defense / Inventory |
| **敌人** | Dummy Beast：巡逻 + 看到玩家后攻击 |
| **AI Profile** | M2: TaskFirst only → M6: +Hunter +Coward |
| **动作** | MoveToResource / PickUpResource / MoveToExit / BiteEnemy / FleeFromThreat / Wait |
| **UI** | Start / Pause / Step / Reset / Save / Load / Brain View / Mission Report |
| **报告** | Success/Failure / Final HP / Resource count / Selected actions / Failure reason |

---

# 4. 风险清单

| 风险 | 应对 |
|------|------|
| AI 看起来像乱跑 | M3 做 Brain View，每个 action 有 reason |
| 系统太复杂没人能调 | 先只给 3 个 AI Profile，不开放自由编辑 |
| 战斗不好玩 | 第一版战斗只验证 AI 策略（打/跑/完成任务），不追求手感 |
| 美术拖住开发 | 全用占位几何体，身体部件只在 HUD 展示 |
| 框架 AI 扩展未到位 | 游戏层先硬编码 Profile Adapter，不等框架补 P0 |

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
| Projectile / Slash / Grapple | 引入弹道/扇形判定/抓取物理，复杂度高 |
| 部件破坏 | 需要部位碰撞+局部功能失效+AI残疾重规划 |
| 群体战斗 / 领地战斗 | 需要小队/仇恨/保护系统 |
| Utility / Social 部件 | 推向小队/协作/生态，复杂度提前爆炸 |

---

# 6. 框架调整 Issue 建议（暂不创建）

待游戏层验证成立后，在 WGameFramework 仓库创建：

| 优先级 | Issue | 内容 |
|--------|-------|------|
| P0 | [AI] Goal Priority Modifier | 给 PriorityGoalSelector 增加 IGoalPriorityModifier 接口 |
| P0 | [AI] Action Cost Modifier | 给 IAiAction 增加 cost modifier 机制 |
| P0 | [AI] PlannerTrace Output | 给 SequentialPlanner 增加 trace 输出 + DebugUI 集成 |
| P1 | [Combat] HazardVolume | 新增 HazardVolume 概念 + tick 伤害/Buff 系统 |
| P1 | [Navigation] Minimal NavGraph | 纯 C# A* + Waypoint/Edge + portal 限制 |
| P2 | [Gameplay] Inventory Component | 通用 Inventory 组件 + hash + SaveState |
