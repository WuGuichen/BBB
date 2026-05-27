# VVV 项目实现方案：AI 生物任务竖切优先

## 0. 制作判断

完整形态设想：

```text
玩家设计生物身体
↓
配置 AI 行为
↓
投放到生态地图
↓
生物自主行动
↓
战斗、逃跑、采集、探索
↓
玩家分析失败原因
↓
继续改造
```

但不能直接做完整生态沙盒。当前最合适的路线是：

```text
先做任务制小地图
再做生态区域
再做身体部件
再做玩家 AI 蓝图
最后才做长期生态/Mod/程序化生成
```

第一个正式目标不是"开放生态沙盒"，而是：

> **Creature Mission Slice：一只可配置 AI 的生物，在一个确定性小地图中完成回收任务。**

四个核心验证问题：

1. 玩家能不能看懂生物为什么这么做？
2. 玩家改 AI 后，行为是否明显变化？
3. 玩家改身体部件后，任务结果是否明显变化？
4. 失败后，玩家是否知道下一步该怎么改？

---

# 1. 当前项目能力评估

## 1.1 已经可以直接复用的能力

### Runtime Foundation

已有 `RuntimeHost`、显式 `RuntimeFrame` / `RuntimeClock`、`RuntimeCommandBuffer`、Replay recorder、Replay playback runner、Runtime result hash、SaveState DTO、SaveState JSON roundtrip 等能力。

制作判断：必须复用。不要另外写一套 GameLoop。不要让 Unity Update 直接驱动玩法权威状态。

AI 生物模拟应该走：

```text
Unity Input / UI / Debug
↓
RuntimeCommandBuffer
↓
RuntimeHost Tick
↓
AI / Physics / Gameplay / Mission modules
↓
Replay hash / SaveState / Diagnostics
```

---

### 属性、Buff、Modifier

属性系统、Buff 系统、Modifier 系统已经是 v1 可用。属性可注册、读取 final value、添加 Add/Mul/FinalMul 修改器、监听变化；Buff 支持生命周期、Tick、堆叠、快照和配置驱动；Modifier 支持条件 + 效果组合。

制作判断：身体部件第一版应该映射成属性、Buff、Modifier。

例如：

```text
厚甲部件
→ MaxHp +30
→ MoveSpeedScale -15%
→ PhysicalResistance +20

嗅觉腔
→ SensorSmellRange +8
→ Unlock fact: corpse.smell.visible

毒刺
→ Unlock action: PoisonSting
→ ApplyBuff: Poison
```

---

### Runtime AI Planner

已有 `IAiWorldState`、`IAiGoal`、`IAiAction`、`IAiCondition`、`IAiEffect`、`SequentialPlanner` 等接口。Planner 规划时 clone 世界状态，不修改传入世界；框架动作的 `Apply` 只修改模拟状态，真实移动、攻击、放技能由游戏层执行。

制作判断：AI 蓝图第一版不要做行为树编辑器、完整 GOAP 或自由脚本。先把玩家 AI 配置映射到 Fact / Goal / Action / Action cost / Goal priority。

第一版玩家并不直接"写 AI"，而是选择 AI 模块（胆小模块、猎手模块、拾荒模块、护卫模块、搬运模块），这些模块实际改 Goal priority / Action cost / Action precondition / Fact projection。

---

### Combat Physics

已落地确定性战斗物理查询和 kinematic motion。使用 `Fix64` / `FixVector3`，核心战斗逻辑不应直接读 Unity Physics。`CombatPhysicsWorld` 当前注册 body 和 AABB collider，查询结果按确定性规则排序。

制作判断：拥抱 AABB / Sphere / Capsule / Sector query。不做三角网格碰撞、复杂 ragdoll、软体、GJK/EPA。

第一版战斗只做：Sphere 感知、AABB 拾取、Sphere Bite、AABB / Capsule Charge、Sector / Sphere 威胁范围、Hazard Volume 持续伤害。

---

### Character Control

已有统一 `CharacterCommand`、`ICharacterCommandSource`、状态机、Combat Motion resolver、Combat/Gameplay action bridge、Gameplay pressure reaction、MxAnimation presentation adapter、Debug UI source、本地输入 command source、Runtime AI Planner command source，以及 playable vertical slice。

`CharacterMotionResolver` 通过 `CombatKinematicMotor` 得到权威移动，不能用 Unity `CharacterController`、`Rigidbody`、`UnityEngine.Physics` 或表现层 root motion 作为权威。

制作判断：Creature 的第一版控制层不要另起炉灶。可以先复用 CharacterControl 思路，但命名上做 Game 层 adapter（CreatureControlAdapter），但不要重写运动管线。

---

### Character Action

`CharacterAction` 代码已合并，包含动作意图请求、动作解析、动作计划、动作执行器、Track Adapter、取消/打断规则、资源依赖、反应上下文、动作诊断、配置定义和配置校验，但目前**代码完成、测试通过，但尚未接入任何运行管线**，`CharacterControl` 仍使用旧 v0 路径。

制作判断：第一阶段不要依赖 CharacterAction。第二阶段可以做试点。第三阶段再正式接入。

---

### Gameplay / Ability

已有 Gameplay Component Runtime：`RuntimeHost` 驱动 spawn、target、ability rules、cleanup、events、hash、SaveState 的最小闭环；Ability 支持 cooldown gate、attribute cost gate、targeting、buff/modifier runtime 等。

配置驱动 Ability 已能用 `BasicAbilityConfig -> RuntimeAbilityConfigResolver -> ConfigAbilityFactory` 创建技能；当前支持 Self、SingleEnemy、DamageByAttackDefense、ApplyBuff。外部 JSON 和后续编辑器不应该直接生成 `BasicAbilityConfig`，而应先生成 `AbilityAuthoringContract`，经过结构化校验后再映射到运行时配置。

制作判断：第一版技能不要做复杂 AbilityGraph。先用 Damage / ApplyBuff / Cooldown / Cost / Targeting。

---

### Debug / Observability

已有 Debug UI、timeline view model、Gameplay/Combat timeline adapters、Entity watch、Performance counters、Simulation Harness reports、Config hot reload Debug UI 等。Debug UI 默认只读，表现状态不进入 Replay、SaveState 或 Runtime hash。

制作判断：这类游戏必须把调试 UI 产品化。第一版就要做"脑内视图"和"战后报告"。如果 AI 行为不可解释，这个玩法会失败。

---

### 资源系统

已有 `ResourceKey`、`ResourceManager`、Catalog、Provider、Variant、RetainPolicy、AssetBundleProvider、RemoteBundleProvider、Preload 等路线。业务配置只保存 `ResourceKey`，不保存 Unity 路径或 `UnityEngine.Object`。

制作判断：MVP 用 ResourcesProvider / MemoryResourceProvider。正式内容用 ResourceKey + Catalog。不要现在接 Addressables、YooAsset 或完整远端热更。

---

### 角色资源包

已有角色资源包 Runtime Spawn 切片，可以通过 `CharacterImportedPackageJson.LoadFromDirectory(...)` 读取导入产物，并通过 `CharacterRuntimeSpawnResolver.Resolve(...)` 或 `CharacterRuntimeSpawnModule` 得到 `CharacterRuntimeBinding`。已有 Iron Vanguard 预览 Prefab、预览场景和 Locomotion Calibration Scene。

制作判断：不要把第一版目标设为"程序化角色建模"。用 Iron Vanguard 或低模占位体先跑通任务。生物身体第一版是数值和碰撞代理，不是模型生成。

---

# 2. 产品定位收敛

## 2.1 不建议当前直接做的完整目标

暂时不要做：

```text
开放世界生态
程序化身体建模
复杂 IK 步态
玩家自由 AI 图编辑器
繁殖/遗传
多人蓝图对战
完整 Mod SDK
动态天气生态演化
```

---

## 2.2 当前最优目标

> **VVV Creature Mission Slice**

核心体验：

```text
玩家在实验室里选择一个生物配置
↓
选择一个 AI 行为配置
↓
投放到小型任务地图
↓
生物自动探索、拾取、战斗、撤退
↓
玩家可用信标做有限干预
↓
任务结束后看到行为解释和失败原因
↓
玩家修改身体或 AI 再试
```

这不是完整游戏，但它是完整的"玩法闭环"。

---

# 3. 高性价比简化策略

## 3.1 身体系统简化

第一版只做：

```text
身体部件改变属性
身体部件改变可用 Action
身体部件改变 Sensor Fact
身体部件改变碰撞代理参数
身体部件改变显示颜色/图标
```

即：

```text
BodyPartDefinition
→ Attribute modifier
→ Buff / Modifier
→ Action unlock
→ Sensor unlock
→ Collider profile
→ Visual token
```

不要先做真实身体拼装。

---

## 3.2 AI 蓝图简化

第一版做：

```text
AI Profile = GoalPolicy + ActionPolicy + FactBinding
```

例如：

```text
胆小拾荒者
- goal.return.priority +30 when inventory.full
- goal.flee.priority +60 when threat.near
- action.attack.cost +30
- action.pickup.cost -10
```

玩家看到的是中文规则：

```text
血量低时更倾向撤退
看到资源时更倾向拾取
敌人强于自己时避免战斗
背包满时返回出口
```

运行时映射成：AiFactKey / IAiGoal priority / IAiAction cost / precondition / effect

---

## 3.3 战斗简化

第一版只做：Bite / Charge / Flee / PickUp / Return

判定：Bite = Sphere query，Charge = Capsule/AABB query + 位移，PickUp = Sphere/AABB query，Flee = 远离威胁方向，Return = 导航到出口

伤害：Combat query 命中 → Gameplay target filter → DamageByAttackDefense → Attribute HP change → Event / Debug log

---

## 3.4 地图简化

第一版只做：Room + Portal / AABB wall / AABB hazard / Sphere resource / Sphere exit / Sphere noise / smell / threat zone

地图不追求自然，追求可测。

---

## 3.5 生态简化

第一版只做：1 个敌人 / 1 个资源 / 1 个出口 / 1 个危险区 / 1 个任务目标

第二版再加：猎物 / 清道夫 / 守卫 / 尸体 / 简单领地

---

## 3.6 UI 简化

第一版建议：UI Toolkit 做全部手测 HUD 和脑内视图，FairyGUI 暂缓。因为第一版主要是制作人/开发者验证，不是玩家商业界面。

---

# 4. 推荐模块划分

不要把游戏逻辑继续塞进 `MxFramework`。框架只提供通用机制，游戏层定义业务 ID 和逻辑。

新增游戏层目录：

```text
Assets/Scripts/VVV/
├── Creature/
├── CreatureAi/
├── CreatureCombat/
├── EcoMap/
├── Mission/
├── Lab/
├── Diagnostics/
├── Demo/
└── Tests/
```

---

## 4.1 Creature 模块

职责：生物定义、身体部件定义、生物实例创建、身体部件转属性/Action/Sensor

核心数据：

```csharp
public sealed class CreatureDefinition
{
    public int CreatureId;
    public string StableId;
    public int CorePartId;
    public int MovementPartId;
    public int SensorPartId;
    public int WeaponPartId;
    public int DefensePartId;
    public int BrainPartId;
    public int AiProfileId;
}

public sealed class BodyPartDefinition
{
    public int PartId;
    public string StableId;
    public BodyPartSlot Slot;
    public int[] AttributeModifierIds;
    public int[] UnlockActionIds;
    public string[] UnlockFactKeys;
    public ColliderProxyProfile ColliderProfile;
    public ResourceKey VisualPrefabKey;
}
```

第一版只需要：Core x1 / Movement x2 / Sensor x1 / Weapon x2 / Defense x1 / Brain x2

---

## 4.2 CreatureAi 模块

职责：把感知和状态转成 IAiWorldState、把 AI Profile 转成 goals/actions/cost、把 planner 输出转成 RuntimeCommand / CharacterCommand、记录 AI 决策日志

核心流程：

```text
Sensor Phase → 写入 AiWorldState facts
Need Phase → hp.low / inventory.full / threat.near / resource.visible
Policy Phase → 根据 AiProfile 调整 goal priority / action cost
Plan Phase → SequentialPlanner.TryPlan
Command Phase → first action → RuntimeCommandBuffer
Trace Phase → DecisionTrace / DebugSource
```

核心类型：

```csharp
public sealed class CreatureAiProfile
{
    public int ProfileId;
    public AiGoalPolicy[] GoalPolicies;
    public AiActionPolicy[] ActionPolicies;
    public AiFactBinding[] FactBindings;
}

public sealed class CreatureAiDecisionTrace
{
    public RuntimeFrame Frame;
    public int CreatureId;
    public string[] Facts;
    public string SelectedGoal;
    public string SelectedAction;
    public string RejectReason;
}
```

---

## 4.3 CreatureCombat 模块

职责：把 AI Action 或玩家间接命令转成 CombatPhysics query、把 query hit 转成 Gameplay damage/buff/effect、输出战斗诊断

第一版动作：Bite / Charge / Flee / Wait / PickUp / Return

不要先做：连招、打断、复杂动作窗口、部位破坏、角色动作新架构接入

---

## 4.4 EcoMap 模块

职责：房间图、碰撞体生成、资源点、出口、危险区、感知区、导航节点、地图语义标签

核心数据：

```csharp
public sealed class EcoRoomDefinition
{
    public int RoomId;
    public string StableId;
    public Aabb Bounds;
    public EcoZoneDefinition[] Zones;
    public EcoPortalDefinition[] Portals;
    public EcoResourceNodeDefinition[] Resources;
    public EcoHazardDefinition[] Hazards;
}
```

第一版地图：单房间实验场 / 1 个资源点 / 1 个出口 / 1 个敌人巡逻区 / 若干 AABB 障碍

---

## 4.5 Mission 模块

职责：任务目标、任务状态、成功/失败判定、任务奖励、战后报告

第一版目标：CollectResource / ReturnToExit / Survive

核心数据：

```csharp
public sealed class MissionDefinition
{
    public int MissionId;
    public string StableId;
    public int MapId;
    public MissionObjectiveDefinition[] Objectives;
    public int MaxFrames;
}
```

---

## 4.6 Diagnostics 模块

职责：AI 脑内视图、任务时间线、战斗事件、物理查询、任务报告、批量模拟报告

必须第一版就做。因为这不是普通动作游戏，玩家的乐趣来自理解 AI 失败原因。

---

# 5. 开发阶段方案

## Milestone 0：项目收敛与切片骨架

目标：建立 VVV 游戏层，不再只做框架 showcase。

要做：
1. 新建 `Assets/Scripts/VVV/` 游戏层
2. 新建 `VVVCreatureMissionSlice` 场景生成菜单
3. 建立 `VVVRuntimeCompositionRoot`
4. 接入 `RuntimeHost`、`RuntimeCommandBuffer`、`CombatPhysicsWorld`、`GameplayComponentWorld` 或 `RuntimeEntity`
5. 建立最小 Debug HUD

产物：

```text
Assets/Scenes/VVV/CreatureMissionSlice.unity
Assets/Scripts/VVV/Demo/CreatureMissionSliceRunner.cs
Assets/Scripts/VVV/Tests/CreatureMissionSliceTests.cs
```

验收：Play 后出现场景、HUD、1 个生物、1 个资源、1 个出口。RuntimeHost 正常 Tick。Reset 后状态一致。Save / Load 能恢复基础状态。

不做：美术、复杂 AI、敌人、战斗、资源系统正式接入

---

## Milestone 1：确定性地图 + 单体移动

目标：一只生物能在 AABB 房间里移动、避障、到达目标。

复用：CombatPhysicsWorld / CombatKinematicMotor / RuntimeCommandBuffer / CharacterControl 思路

要做：
1. `EcoMapDefinition`
2. `EcoRoomDefinition`
3. `EcoObstacleDefinition`
4. `EcoNavigationGraph`
5. `CreatureMotionAdapter`
6. `MoveToTargetCommand`
7. `CreatureViewSync`

地形限制：只允许平面 AABB 房间、只允许 AABB 障碍、不做楼梯/斜坡/动态障碍

验收：生物从起点走到资源点。撞墙不会穿透。重复运行 Replay hash 一致。不同机器/不同帧率不改变结果。

测试：MoveToTarget reaches target / MoveIntoWall stops at wall / Replay same command stream produces same final position / SaveState restore continues movement

---

## Milestone 2：AI Planner 单体闭环

目标：生物能根据事实和目标自主选择行动。

复用：Runtime AI Planner / AiWorldState / AiFactKey / IAiGoal / IAiAction / SequentialPlanner

要做：
1. `CreatureFactKeys`
2. `CreatureAiWorldStateProjector`
3. `CreatureGoalFactory`
4. `CreatureActionFactory`
5. `CreatureAiProfile`
6. `CreatureAiDecisionTrace`
7. `CreatureBrainDebugSource`

第一版事实：resource.visible / exit.visible / inventory.full / hp.low / enemy.visible / enemy.near / threat.near

第一版目标：collect.resource / return.exit / avoid.death / defeat.enemy / idle.safe

第一版动作：MoveToResource / PickUpResource / MoveToExit / FleeFromThreat / BiteEnemy / Wait

验收：没有敌人时，生物拾取资源并返回出口。出现敌人且 hp.low 时，生物逃跑。AI Debug HUD 能显示当前 facts、selected goal、selected action。

关键简化：AI 只每隔 N 帧决策一次。Action 执行中不每帧重规划。只执行 plan 的第一个 action。避免 AI 抖动。

---

## Milestone 3：拾取任务闭环

目标：形成第一个真正可玩的任务。

要做：
1. `MissionDefinition`
2. `MissionRuntimeModule`
3. `InventoryComponent`
4. `ResourceNodeComponent`
5. `ExitZoneComponent`
6. `MissionResult`
7. `BattleReport / MissionReport`

任务流程：Spawn creature → Spawn resource nodes → AI picks resource → AI returns to exit → Mission success → Report generated

报告内容：成功/失败、耗时 frame、拾取资源数、受伤次数、触发规则次数、最后 10 个 AI 决策、失败原因、建议修改

验收：玩家不操作时，默认 AI 可完成简单任务。切换为"攻击型 AI"后，生物更容易追敌偏离目标。切换为"胆小型 AI"后，生物更容易撤退。

第一版玩家交互：Start Mission / Pause / Step Frame / Reset / Save / Load / Toggle Brain View

---

## Milestone 4：基础战斗

目标：敌人出现后，生物能战斗、受伤、死亡或撤退。

复用：CombatPhysicsWorld.Query / Gameplay attribute / buff / Config Driven Ability / Gameplay Diagnostic Snapshot

要做：
1. `CreatureCombatRuntimeModule`
2. `BiteAction`
3. `ChargeAction`
4. `DamageResolver`
5. `HealthDeathSystem`
6. `CreatureTeamRelation`
7. `CombatTimelineDebugSource`

攻击判定：Bite = Sphere query / Charge = Capsule/AABB sweep 简化

伤害结算：Attack - Defense / 最低伤害 1 / HP <= 0 → Dead / Dead entity → PendingDestroy or corpse

验收：玩家生物能咬死弱敌。强敌能杀死玩家生物。战报能说明死亡原因。重复 Replay 结果一致。

不做：连招、打断、动作帧窗口、部位破坏、复杂目标选择

---

## Milestone 5：身体部件 v0

目标：玩家可以通过身体部件影响任务结果。

要做：
1. `BodyPartDefinitionConfig`
2. `CreatureBuildDefinition`
3. `CreatureBuildResolver`
4. `PartAttributeModifierApplier`
5. `PartActionUnlockResolver`
6. `PartSensorUnlockResolver`
7. `CreatureBuildValidationReport`

第一版部件：

```text
Core: light_core / heavy_core
Movement: fast_legs / stable_legs
Sensor: basic_eye / smell_sensor
Weapon: bite_jaw / charge_horn
Defense: thin_skin / armor_shell
Brain: primitive_brain / tactical_brain
```

效果示例：

```text
heavy_core: +MaxHp / -MoveSpeed
smell_sensor: unlock fact corpse.near / resource.smell
charge_horn: unlock action Charge
tactical_brain: max goals +1 / decision interval -1
```

验收：fast_legs 构筑更快完成回收任务。armor_shell 构筑更容易活过战斗。smell_sensor 构筑能在无视野时找到资源。charge_horn 构筑能击退敌人。

性价比简化：模型暂不变。只改颜色、图标、HUD 文案。

---

## Milestone 6：AI Profile v0

目标：玩家可以选择不同 AI 性格，而不需要编辑复杂图。

要做：
1. `AiProfileDefinition`
2. `GoalPriorityModifier`
3. `ActionCostModifier`
4. `ConditionOverride`
5. `AiProfileRuntimeResolver`
6. `AiProfileDebugView`

第一版 Profile：Scavenger / Hunter / Coward / Guardian / TaskFirst

示例：

```text
Scavenger: collect.resource +30 / attack.enemy cost +20 / flee.threat +10
Hunter: defeat.enemy +40 / bite.enemy cost -10 / return.exit -10
Coward: avoid.death +60 / attack.enemy cost +40
TaskFirst: collect.resource +50 / return.exit +40 / investigate.noise cost +30
```

验收：同一个身体，换不同 Profile 后任务行为明显不同。Brain View 能解释 goal priority 和 action cost 的变化。

注意：这一步不是玩家 AI 编辑器，只是 AI 模块选择。

---

## Milestone 7：生态地图 v0

目标：地图不只是场地，而是带语义的生态容器。

要做：
1. `HabitatZoneDefinition`
2. `HazardZoneDefinition`
3. `SpawnProfileDefinition`
4. `DenPointDefinition`
5. `EcoMapRuntimeModule`
6. `SpeciesSpawnResolver`

地图标签：light.dark / terrain.open / terrain.narrow / hazard.acid / resource.bio / resource.metal / danger.low / danger.high

物种分布：spawn score = terrain match + resource match + den match - hazard mismatch - predator pressure

第一版物种：prey_bug / scavenger_bug / guard_beast

验收：prey_bug 出现在开阔资源区。scavenger_bug 会靠近尸体。guard_beast 守着高价值资源点。不同地图标签改变 spawn 结果。

性价比简化：不做长期生态演化、不做繁殖、不做迁徙。只做 mission start 时的生态生成。

---

## Milestone 8：玩家间接指挥

目标：玩家能影响 AI，但不能直接操控 AI。

要做：
1. `PlayerSignalCommand`
2. `BeaconZone`
3. `RecallSignal`
4. `ThreatMarkSignal`
5. `ResourceMarkSignal`
6. `SignalFactProjector`

第一版信号：Move Beacon / Recall Beacon / Threat Mark / Resource Mark

映射方式：

```text
Recall Beacon → fact signal.recall = true → goal.return.exit priority +80
Threat Mark → fact marked.enemy = target → goal.defeat.enemy +30 或 goal.avoid.threat +30（取决于 AI Profile）
```

验收：TaskFirst AI 会响应 Recall。Hunter AI 会响应 Threat Mark。Coward AI 可能把 Threat Mark 当作避开区域。

---

## Milestone 9：战后分析产品化

目标：让玩家知道为什么失败，以及该怎么改。

要做：
1. `MissionTimeline`
2. `AiDecisionTimeline`
3. `CombatEventTimeline`
4. `FailureClassifier`
5. `BuildRecommendationEngine`
6. `MissionReportUi`

失败分类：died.to.enemy / died.to.hazard / failed.timeout / lost.path / ignored.objective / overaggressive / overcoward / low.sensor / low.damage / low.mobility

建议系统第一版硬编码规则：

```text
death.to.hazard 且 hazard.acid → 推荐 acid_resistance 或 avoid_hazard rule
ignored.objective → 推荐 TaskFirst profile
overaggressive → 推荐 Coward / TaskFirst 或提高 attack cost
low.sensor → 推荐 smell_sensor 或 night_eye
```

验收：每次失败都有明确原因。每次报告能指出至少一个可改构筑点。玩家能根据报告下一局改出不同结果。

---

# 6. 后续阶段

## 6.1 CharacterAction 集成

等基础任务和战斗成立后，再把 CharacterAction 接入。

接入点：CreatureAi selected action → CharacterActionIntentRequest → CharacterActionResolver → CharacterActionPlan → CharacterActionRunner → CharacterActionTrackAdapter → Motion / Combat / Gameplay / Animation

目前未接入运行管线，不要作为 MVP 依赖。

---

## 6.2 正式资源包

路线：ResourceKey → Catalog → AssetBundleProvider → Preload group → Runtime diagnostics

不要把 Prefab 直接塞业务配置。业务配置只保存 `ResourceKey`。

---

## 6.3 Player-facing UI

当前阶段用 UI Toolkit 做 debug 和验证。后续商业化 UI 再迁移 FairyGUI。

---

## 6.4 程序化身体 / 程序化动画

先不做。替代方案：低模占位体、颜色区分、图标区分、简单 scale、简单 attachment、现有 locomotion clips / MxAnimation。

---

# 7. 技术总架构

## 7.1 Runtime Loop

```text
Unity Input / UI
↓
RuntimeCommandBuffer
↓
RuntimeHost Tick
↓
MissionRuntimeModule
↓
EcoMapRuntimeModule
↓
CreatureSensorModule
↓
CreatureAiRuntimeModule
↓
CreatureCommandModule
↓
CreatureMotionModule
↓
CreatureCombatModule
↓
GameplayRuntimeModule
↓
Diagnostics / Replay / Hash / SaveState
```

关键原则：Unity 只做输入、显示、资源、UI。权威逻辑全部走 Runtime。

---

## 7.2 单个生物运行流程

```text
1. Sensor 收集世界信息
2. Projector 写入 AiWorldState
3. Profile 调整目标优先级和动作成本
4. SequentialPlanner 规划
5. 取第一个 action
6. 转成 CreatureCommand
7. Movement / Combat / Mission 执行
8. 写事件、hash、trace
9. Debug UI 读取快照
```

---

## 7.3 生物数据流

```text
CreatureBuildDefinition
↓
CreatureBuildResolver
↓
AttributeStore / GameplayAttributeSet
↓
BodyPart action unlocks
↓
Sensor fact bindings
↓
Collider proxy profile
↓
Visual binding
```

---

# 8. 具体数据配置建议

## 8.1 BodyPartConfig

```json
{
  "id": 1001,
  "stableId": "part.movement.fast_legs",
  "slot": "Movement",
  "displayName": "Fast Legs",
  "attributeModifiers": [
    { "attribute": "move.speed", "phase": "Mul", "value": 1.25 },
    { "attribute": "stability", "phase": "Add", "value": -10 }
  ],
  "unlockActions": ["action.move", "action.flee"],
  "unlockFacts": ["fact.self.can_run"],
  "colliderProfile": "collider.small_quadruped",
  "visualKey": "vvv.part.fast_legs"
}
```

## 8.2 AiProfileConfig

```json
{
  "id": 2001,
  "stableId": "ai.profile.task_first",
  "goalPolicies": [
    { "goal": "goal.collect_resource", "priorityAdd": 50 },
    { "goal": "goal.return_exit", "priorityAdd": 40 }
  ],
  "actionPolicies": [
    { "action": "action.attack_enemy", "costAdd": 20 },
    { "action": "action.pickup_resource", "costAdd": -10 }
  ]
}
```

## 8.3 MissionConfig

```json
{
  "id": 3001,
  "stableId": "mission.lab_collect_01",
  "mapId": 4001,
  "objectives": [
    { "type": "CollectResource", "count": 1 },
    { "type": "ReturnToExit" }
  ],
  "maxFrames": 7200
}
```

## 8.4 EcoMapConfig

```json
{
  "id": 4001,
  "stableId": "map.lab_room_01",
  "rooms": [
    {
      "id": 1,
      "bounds": { "min": [-8, 0, -8], "max": [8, 3, 8] },
      "tags": ["terrain.open", "light.normal"],
      "hazards": [],
      "resources": ["resource.bio_sample"]
    }
  ]
}
```

---

# 9. 测试方案

## 9.1 单元测试

必须覆盖：BodyPartResolver / AiWorldStateProjector / AiProfilePolicyResolver / SequentialPlanner action selection / Combat query hit result / Mission objective state / FailureClassifier

示例：给定 hp.low=true, threat.near=true，当 profile=Coward，则 selected goal=avoid.death，且 first action=FleeFromThreat

---

## 9.2 集成测试

必须覆盖：Creature can collect resource and exit / Creature flees when hp low / Creature attacks when Hunter profile / Creature ignores enemy when TaskFirst profile / Replay same command stream equals same hash / Save/Load restore mission state

---

## 9.3 PlayMode 测试

必须覆盖：CreatureMissionSlice scene opens / HUD labels non-empty / Start/Reset/Pause buttons work / Brain View shows selected action / Mission Report appears after success/failure

UI Toolkit 可玩 Demo 的验收不应只检查 visual tree 节点存在，还要读取关键 Label/Button 的 text 和 resolvedStyle.color，确保文本非空且 alpha > 0。

---

## 9.4 批量仿真

应建立：100 次 TaskFirst profile collect mission / 100 次 Hunter profile collect mission / 100 次 Coward profile enemy mission / 100 次 fast legs vs armor shell comparison

输出：成功率、平均帧数、死亡率、击杀率、拾取率、逃跑次数、AI replan 次数、失败分类分布

---

# 10. 第一批 GitHub Issue 建议

## Epic 1：Creature Mission Slice

```text
[VVV] Create CreatureMissionSlice scene generator
[VVV] Add VVVRuntimeCompositionRoot
[VVV] Wire RuntimeHost + CommandBuffer + SaveState for CreatureMissionSlice
[VVV] Add UI Toolkit debug HUD shell for CreatureMissionSlice
```

## Epic 2：EcoMap v0

```text
[VVV.EcoMap] Add EcoMapDefinition and single-room map config
[VVV.EcoMap] Generate AABB obstacles into CombatPhysicsWorld
[VVV.EcoMap] Add resource node and exit zone
[VVV.EcoMap] Add minimal navigation graph
```

## Epic 3：Creature v0

```text
[VVV.Creature] Add CreatureDefinition and BodyPartDefinition
[VVV.Creature] Spawn creature with attributes and Combat body
[VVV.Creature] Add collider proxy profile
[VVV.Creature] Add simple visual sync
```

## Epic 4：AI v0

```text
[VVV.AI] Add CreatureFactKeys and world state projector
[VVV.AI] Add basic goals and actions
[VVV.AI] Add AiProfileDefinition and policy resolver
[VVV.AI] Add planner command adapter
[VVV.AI] Add decision trace and Brain Debug HUD
```

## Epic 5：Mission v0

```text
[VVV.Mission] Add collect resource objective
[VVV.Mission] Add return exit objective
[VVV.Mission] Add mission result and report
[VVV.Mission] Add failure classifier v0
```

## Epic 6：Combat v0

```text
[VVV.Combat] Add Bite action using Sphere query
[VVV.Combat] Add Charge action using simple sweep
[VVV.Combat] Apply damage through Gameplay attributes
[VVV.Combat] Add death and corpse state
```

## Epic 7：Testing

```text
[VVV.Tests] Add deterministic replay test for CreatureMissionSlice
[VVV.Tests] Add AI planner profile tests
[VVV.Tests] Add mission success/failure tests
[VVV.Tests] Add batch simulation report for first map
```

---

# 11. 优先级排序

## 必须先做

```text
Runtime slice
单房间地图
单体移动
AI 规划
拾取资源
返回出口
战后报告
Replay/Save/Hash
```

## 第二优先级

```text
敌人
咬击
死亡
身体部件属性
AI Profile
脑内视图
```

## 第三优先级

```text
生态标签
多物种
危险地形
间接信标
批量仿真
```

## 暂缓

```text
程序化建模
复杂动画
玩家 AI 图编辑器
长期生态演化
繁殖
Mod SDK
正式 UI
多人
```

---

# 12. 最小竖切内容清单

## 地图

```text
Lab Room 01
- 1 个封闭房间
- 3 个 AABB 障碍
- 1 个资源点
- 1 个出口
- 1 个敌人出生点
```

## 生物

```text
Player Creature
- HP
- MoveSpeed
- Attack
- Defense
- Inventory
```

## 敌人

```text
Dummy Beast
- 巡逻
- 看到玩家后攻击
```

## AI Profile

```text
TaskFirst
Hunter
Coward
```

## 动作

```text
MoveToResource
PickUpResource
MoveToExit
BiteEnemy
FleeFromThreat
Wait
```

## UI

```text
Start
Pause
Step
Reset
Save
Load
Brain View
Mission Report
```

## 报告

```text
Success / Failure
Final HP
Resource count
Selected actions
Triggered facts
Failure reason
Suggested change
```

---

# 13. 关键制作风险

## 风险 1：AI 看起来像乱跑

解决：第一版 Brain View 必须做。每个 action 都必须有 reason。每个 rejected action 都必须有 reject reason。

## 风险 2：系统太复杂导致没人能调

解决：第一版只有 3 个 AI Profile。不开放自由编辑。不做复杂权重曲线。

## 风险 3：战斗不好玩

解决：第一版战斗不追求手感。战斗只验证 AI 策略：打、跑、绕、完成任务。

## 风险 4：美术拖住开发

解决：第一版只用占位几何体。身体部件只在 HUD 上展示。第二版再换低模 prefab。

## 风险 5：新架构整合风险

解决：CharacterAction 暂缓。先用 CharacterControl v0 + CombatPhysicsWorld + Gameplay Ability。等任务闭环成立后再评估 CharacterAction 集成。

---

# 14. 最终判断

当前项目**不缺框架能力**，反而已经有很多底层能力。真正要避免的是继续横向扩框架，而没有形成游戏闭环。

当前最正确的制作路线是：

```text
不要先做完整 AI 生物沙盒
不要先做程序化建模
不要先做完整生态
不要先做玩家 AI 编辑器

先做：
Creature Mission Slice
```

这条切片要证明：

```text
AI 能规划
生物能执行
地图能约束
任务能成功或失败
玩家能理解原因
玩家能改配置影响下一局
```

一旦这个闭环成立，再扩：

```text
身体部件
生态地图
多物种
玩家 AI 蓝图
正式美术
Mod
```

否则任何后续功能都会变成理想假设。

> **第一目标：一只 AI 生物能在确定性小地图里自主完成回收任务，并生成可解释战报。**

这就是 VVV 从"框架项目"转向"游戏项目"的最小拐点。
