# TECHNICAL — 技术方案与框架映射

> 版本 0.2 | 2026-05-27
>
> 本文描述游戏系统如何映射到 MxFramework，以及框架的缺口和调整建议。
> 玩法设计见 [GDD.md](GDD.md)，制作计划见 [MILESTONES.md](MILESTONES.md)。

---

# 1. 框架能力总览

基于 `WGameFramework/Docs/CAPABILITIES.md`（2026-05-24）校准。

| 框架系统 | 成熟度 | 游戏层用途 |
|----------|--------|-----------|
| **Runtime Foundation** | ✅ v0.1 | 主循环、命令缓冲、Replay/Hash、SaveState |
| **Attributes** | ✅ v1 | 生物属性（HP/攻击/移速/感知范围） |
| **Buffs** | ✅ v1 | 持续效果（中毒/加速/减速/恐惧） |
| **Modifiers** | ✅ v1 | 部件加成、AI 状态修正 |
| **Config** | ✅ v1 | 部件/AI/任务/地图配置 |
| **Config Factories** | ✅ v1 | 配置驱动创建 Buff/Modifier/Ability |
| **Runtime AI Planner** | ✅ v1 | 生物 AI 行为规划（GOAP 风格） |
| **Combat Physics** | ✅ v1 | 碰撞查询（Ray/AABB/Sphere/Capsule/Sector） |
| **Combat Motion** | ✅ v0.1 | 固定帧移动、重力、AABB 阻挡 |
| **Character Control** | ✅ v0.1-v0.6 | 统一角色命令、状态机、Motion resolver |
| **Gameplay** | ✅ v0.1-v0.2 | 实体、技能、目标选择、冷却、组件运行时 |
| **Gameplay Component** | ✅ v0.1 | 组件式实体（Spawn/Cleanup/Hash/SaveState） |
| **Resources** | ✅ v0.5 | ResourceKey / Catalog / Provider / Preload |
| **Input** | ✅ v0.1 | 输入采集、意图命令、上下文栈 |
| **Debug UI** | ✅ v0.1-v0.2 | 调试源注册、Timeline、Entity Watch |
| **MxAnimation** | ✅ v0.1-v0.4 | 动画表现层（Playables/Blend/Bake） |
| **Character Action** | 🟡 代码完成 | 动作系统（8391行，未接入运行管线） |

---

# 2. 系统映射

## 2.1 身体部件 → Attributes / Modifiers / Config

映射规则：每个 `BodyPartDefinition` 转为属性修改器 + Action 解锁 + Fact 解锁。

```
厚甲部件  →  MaxHp +30, MoveSpeedScale -15%, PhysicalResistance +20
嗅觉腔    →  SensorSmellRange +8, Unlock fact: corpse.smell.visible
毒刺      →  Unlock action: PoisonSting, ApplyBuff: Poison
战术脑    →  max goals +1, decision interval -1
```

不做的：真实骨骼拼装、程序化建模、部件物理连接。第一版身体是数值和碰撞代理。

## 2.2 AI Profile → Runtime AI Planner

映射规则：`AiProfile` = Goal priority 偏移 + Action cost 偏移 + Fact binding。

运行时流程：

```
Sensor Phase  → 收集世界信息，写入 AiWorldState facts
Need Phase    → hp.low / inventory.full / threat.near / resource.visible
Policy Phase  → AiProfile 调整 goal priority / action cost
Plan Phase    → SequentialPlanner.TryPlan
Command Phase → 取第一个 action → RuntimeCommandBuffer
Trace Phase   → 记录 DecisionTrace（⚠️ 需框架补能力）
```

## 2.3 战斗 → Combat Physics + Gameplay Ability

| 游戏攻击 | 框架实现 |
|----------|----------|
| Bite 咬击 | `CombatPhysicsWorld.Query(Sphere)` + `DamageByAttackDefense` |
| Charge 冲撞 | `CombatKinematicMotor.MoveForward` + `Query(Capsule sweep)` + 击退 |
| Slash 爪击 | `Query(Sector)` + `DamageByAttackDefense` |
| Projectile 骨刺 | `CombatPhysicsWorld` Sphere projectile + `ApplyBuff(Poison)` |
| Hazard 酸池 | ⚠️ 需框架补 HazardVolume 或游戏层自建 |

## 2.4 地图 → Combat Physics + NavGraph

| 地图元素 | 框架实现 |
|----------|----------|
| 墙壁 | `CombatPhysicsWorld` AABB body + collider |
| 危险区 | ⚠️ 需 HazardVolume（AABB/Sphere + damagePerTick） |
| 资源点 | Game layer component（Sphere trigger zone） |
| 出口 | Game layer component（Sphere trigger zone） |
| 寻路 | ⚠️ 需纯 C# NavGraph（Waypoint + Edge + A*） |

## 2.5 移动 → Combat Kinematic Motor

生物移动复用 `CombatKinematicMotor`：固定帧移动、重力（如有）、AABB 阻挡。

不同生物通过 Motor 参数区分：MaxSpeed、Acceleration、TurnRate、Capsule radius。

## 2.6 主循环 → Runtime Host

```
Unity Input / UI
    ↓
RuntimeCommandBuffer
    ↓
RuntimeHost Tick
    ├── MissionRuntimeModule        (游戏层)
    ├── EcoMapRuntimeModule          (游戏层)
    ├── CreatureSensorModule         (游戏层)
    ├── CreatureAiRuntimeModule      (游戏层)
    ├── CreatureCommandModule        (游戏层)
    ├── CreatureMotionModule         (复用 CombatKinematicMotor)
    ├── CreatureCombatModule         (游戏层 + CombatPhysicsWorld)
    ├── GameplayRuntimeModule        (复用框架)
    └── Diagnostics / Replay / Hash / SaveState (复用框架)
```

关键原则：Unity 只做输入、显示、资源、UI。权威逻辑全部走 Runtime。

## 2.7 配置 → Config System

所有游戏配置用 `ConfigTable<T>` + `ConfigSchema`：

| 配置类型 | 对应 |
|----------|------|
| BodyPartConfig | 部件属性、解锁 Action、解锁 Fact、碰撞代理、视觉 key |
| AiProfileConfig | Goal priority 偏移、Action cost 偏移 |
| MissionConfig | 地图 ID、目标列表、最大帧数 |
| EcoMapConfig | Room 定义、Hazard、Resource、Portal |
| SpeciesConfig | 生态位、偏好标签、Spawn 预算 |

---

# 3. 框架缺口分析

## 3.1 缺口总表

| # | 系统 | 缺口 | 游戏影响 | 优先级 |
|---|------|------|----------|--------|
| A | AI Planner | Goal Priority 无动态调整 | AI Profile 无法生效 | **P0** |
| B | AI Planner | Action Cost 无 modifier 机制 | 性格无法影响行为偏好 | **P0** |
| C+K | AI Planner + Debug | 无 PlannerTrace 输出 | 脑内视图和战后报告做不了 | **P0** |
| E+F | Combat | 无 HazardVolume 概念 | 酸池/毒气需要游戏层自建 | **P1** |
| H | Navigation | 无 NavGraph / A* 寻路 | 生物无法自动避障寻路 | **P1** |
| J | Gameplay | 无 Inventory 组件 | 拾取/携带需要游戏层自建 | **P2** |

## 3.2 P0 缺口详解

### 缺口 A：Goal Priority 动态调整

**需求**：同一身体，不同 AI Profile 改变 goal 优先级。Coward 的 `avoid.death` +60，Hunter 的 `defeat.enemy` +40。

**现状**：`PriorityGoalSelector` 按固定优先级选择目标，无运行时偏移接口。

**建议**：给 `PriorityGoalSelector` 增加 `IGoalPriorityModifier` 接口。

```csharp
public interface IGoalPriorityModifier
{
    int ModifyPriority(IAiGoal goal, int basePriority, IAiWorldState state);
}
```

游戏层注入 AiProfile 的 modifier，运行时按 `(basePriority + modifiers)` 排序。

### 缺口 B：Action Cost 动态调整

**需求**：AI Profile 改变 action 成本。Hunter 的 `bite.cost -10`，Coward 的 `attack.cost +40`。

**现状**：`IAiAction.Cost` 是固定值，无 modifier 机制。

**建议**：给 `IAiAction` 增加 cost modifier（类似 `AttributeModifier`）。

```csharp
public interface IAiActionCostModifier
{
    int ModifyCost(IAiAction action, int baseCost, IAiWorldState state);
}
```

### 缺口 C+K：PlannerTrace

**需求**：战后报告需要结构化 AI 决策历史 — frame、facts、选中的 goal/action、拒绝原因。

**现状**：`SequentialPlanner` 只输出最终 plan，不记录中间决策过程。

**建议**：给 `SequentialPlanner` 增加 trace 输出。

```csharp
public sealed class AiPlannerTrace
{
    public RuntimeFrame Frame;
    public Dictionary<string, object> FactsSnapshot;
    public AiGoalEval[] EvaluatedGoals;    // goal + priority + selected?
    public AiActionEval[] EvaluatedActions; // action + cost + precondition pass/reject reason
    public string SelectedGoal;
    public string SelectedAction;
}
```

接入 `DebugUiTimelineViewModel`，游戏层收集 trace 序列格式化为战后报告。

## 3.3 P1 缺口详解

### 缺口 E+F：HazardVolume

**需求**：酸池、毒气区每 tick 对区域内实体造成伤害/挂 Buff，进入/离开时触发事件。

**建议**：新增 `HazardVolume` 概念。

```csharp
public sealed class HazardVolume
{
    public int VolumeId;
    public Aabb Bounds;           // 或 Sphere
    public int DamagePerTick;     // 0 = 无直接伤害
    public int BuffId;            // 0 = 不挂 Buff
    public int TickInterval;      // 每 N tick 触发一次
}
```

作为 `IRuntimeModule` 接入 RuntimeHost，每 tick 查询 `CombatPhysicsWorld` 区域内实体，自动施加伤害/Buff。

### 缺口 H：NavGraph 寻路

**需求**：生物从 A 走到 B 需要避障寻路（Room + Portal 结构）。

**建议**：新增纯 C# `NavGraph`。

```csharp
public sealed class NavGraph
{
    public NavWaypoint[] Waypoints;
    public NavEdge[] Edges;       // 带 cost 和 portal 限制
}

public static class NavPathfinder
{
    public static NavPath FindPath(NavGraph graph, int from, int to);
}
```

A* 或 BFS，不依赖 Unity NavMesh。游戏层定义 Room/Portal 拓扑，框架提供寻路算法。

## 3.4 P2 缺口详解

### 缺口 J：Inventory Component

**需求**：生物能携带资源（拾取、持有、放下）。

**建议**：新增 `GameplayInventoryComponent`，接入 `GameplayComponentWorld` 的 hash 和 SaveState。

```csharp
public sealed class GameplayInventoryComponent
{
    public int MaxCapacity;
    public int CurrentCount;
    public void Add(int count);
    public void Remove(int count);
    public bool IsFull { get; }
}
```

---

# 4. 框架复用与自建决策

| 游戏需求 | 决策 | 理由 |
|----------|------|------|
| 主循环 | **复用** RuntimeHost + CommandBuffer | 框架核心能力，直接用 |
| 属性系统 | **复用** AttributeStore | v1 稳定，直接用 |
| Buff 系统 | **复用** BuffPipeline | v1 稳定，直接用 |
| AI 规划 | **复用** SequentialPlanner | v1 可用，但需补 3 个 P0 扩展 |
| 碰撞查询 | **复用** CombatPhysicsWorld | v1 稳定，直接用 |
| 移动 | **复用** CombatKinematicMotor | v0.1 可用 |
| 实体管理 | **复用** GameplayComponentWorld | v0.1 可用 |
| 配置 | **复用** ConfigTable + ConfigSchema | v1 稳定，直接用 |
| 回放/存档 | **复用** Replay + SaveState | v0.1 可用 |
| Debug UI | **复用** DebugSourceRegistry + Timeline | v0.1-v0.2 可用 |
| 任务系统 | **自建** MissionRuntimeModule | 框架无通用 Mission 系统，纯游戏层逻辑 |
| 导航 | **自建或框架补** NavGraph | 框架无寻路，建议框架提供最小实现 |
| 危险区 | **自建或框架补** HazardVolume | 框架无持续区域伤害，建议框架提供原语 |
| 生态模拟 | **自建** EcoMapRuntimeModule | 纯游戏层逻辑 |
| 战后报告 | **自建** Diagnostics 模块 | 游戏层格式化，复用框架 trace 输出 |
| 角色控制 | **包 adapter** CreatureControlAdapter | 复用 CharacterControl 思路，不重写运动管线 |

---

# 5. 游戏层模块划分

```
Assets/Scripts/VVV/
├── Creature/              # 生物定义、身体部件、实例创建
├── CreatureAi/            # AI Profile、Fact/Goal/Action 映射、Decision Trace
├── CreatureCombat/        # 攻击判定、伤害结算、死亡
├── EcoMap/                # 房间图、碰撞体、资源点、出口、危险区
├── Mission/               # 任务目标、状态、成功/失败、战后报告
├── Lab/                   # 实验站 UI、部件仓库、AI 蓝图台
├── Diagnostics/           # 脑内视图、时间线、批量仿真报告
├── Demo/                  # 场景入口、组合根
└── Tests/                 # 单元测试、集成测试、PlayMode 测试
```

游戏层不放在 `MxFramework/` 下。框架只提供通用机制，游戏层定义业务 ID 和逻辑。

---

# 6. 运行时管线

单个生物每帧流程：

```
1. Sensor 收集世界信息 → 写入 AiWorldState facts
2. AiProfile 调整 goal priority / action cost（⚠️ 需框架补 A/B）
3. SequentialPlanner 规划 → 输出 plan + trace（⚠️ 需框架补 C）
4. 取第一个 action → 转成 CreatureCommand
5. Movement / Combat / Mission 执行
6. 写事件、hash、trace
7. Debug UI 读取快照
```

关键简化：

- AI 只每隔 N 帧决策一次（避免抖动）
- Action 执行中不每帧重规划
- 只执行 plan 的第一个 action
