# TECHNICAL — 技术方案与框架映射

> 版本 0.3 | 2026-05-27
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

| 游戏攻击 | 框架实现 | 阶段 |
|----------|----------|------|
| Bite 咬击 | `CombatPhysicsWorld.Query(Sphere)` + `DamageByAttackDefense` | `MVP` |
| Charge 冲撞 | `CombatKinematicMotor.MoveForward` + `Query(Capsule sweep)` + 击退 | `MVP` |
| Slash 爪击 | `Query(Sector)` + `DamageByAttackDefense` | `Post-MVP` |
| Projectile 骨刺 | `CombatPhysicsWorld` Sphere projectile + `ApplyBuff(Poison)` | `Long-term` |
| Hazard 酸池 | ⚠️ 需 HazardVolume 或游戏层自建 | `Post-MVP` |

## 2.4 地图 → Combat Physics + NavGraph

| 地图元素 | 框架实现 | 阶段 |
|----------|----------|------|
| 墙壁 | `CombatPhysicsWorld` AABB body + collider | `MVP` |
| 资源点 | Game layer component（Sphere trigger zone） | `MVP` |
| 出口 | Game layer component（Sphere trigger zone） | `MVP` |
| 危险区 | ⚠️ 需 HazardVolume（AABB/Sphere + damagePerTick） | `Post-MVP` |
| 寻路 | MVP: 手工 Waypoint + BFS；Post-MVP: A* + NavGraph | `MVP` |

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

| # | 系统 | 缺口 | 游戏影响 | 优先级 | 策略 |
|---|------|------|----------|--------|------|
| A | AI Planner | Goal Priority 无动态调整 | AI Profile 无法生效 | **P0** | 游戏层先硬编码 |
| B | AI Planner | Action Cost 无 modifier 机制 | 性格无法影响行为偏好 | **P0** | 游戏层先硬编码 |
| C+K | AI Planner + Debug | 无 PlannerTrace 输出 | 脑内视图和战后报告做不了 | **P0** | 游戏层先自建 trace |
| E+F | Combat | 无 HazardVolume 概念 | 酸池/毒气需要游戏层自建 | **P1** | 游戏层自建 |
| H | Navigation | 无 NavGraph / A* 寻路 | 生物无法自动避障寻路 | **P1** | MVP 用手工 Waypoint+BFS |
| J | Gameplay | 无 Inventory 组件 | 拾取/携带需要游戏层自建 | **P2** | 游戏层自建 |

## 3.2 核心策略：游戏层先硬编码验证

**当功能只服务 WGame_B 当前切片时，优先游戏层实现。当两个以上项目或框架 Demo 都需要时，再上升为框架能力。**

具体做法：

### 缺口 A/B：AI Profile Adapter

不等框架补 Goal Priority Modifier 和 Action Cost Modifier。游戏层先做 `CreatureAiProfileRuntimeAdapter`：

```csharp
public sealed class CreatureAiProfileRuntimeAdapter
{
    // 根据 Profile 生成不同的 goals 列表
    public IAiGoal[] CreateGoals(AiProfileKind profile)
    {
        return profile switch
        {
            AiProfileKind.TaskFirst => new IAiGoal[]
            {
                new FactGoal("collect.resource", priority: 90),
                new FactGoal("return.exit", priority: 80),
                new FactGoal("avoid.death", priority: 50),
            },
            AiProfileKind.Hunter => new IAiGoal[]
            {
                new FactGoal("defeat.enemy", priority: 90),
                new FactGoal("collect.resource", priority: 40),
                new FactGoal("return.exit", priority: 30),
            },
            AiProfileKind.Coward => new IAiGoal[]
            {
                new FactGoal("avoid.death", priority: 95),
                new FactGoal("return.exit", priority: 70),
                new FactGoal("collect.resource", priority: 30),
            },
            _ => throw new ArgumentOutOfRangeException(nameof(profile))
        };
    }

    // 根据 Profile 生成不同的 actions 列表
    public IAiAction[] CreateActions(AiProfileKind profile)
    {
        var actions = new List<IAiAction>();
        // 基础 action 所有 profile 共享
        actions.Add(new MoveToResourceAction(cost: 10));
        actions.Add(new PickUpResourceAction(cost: 5));
        actions.Add(new MoveToExitAction(cost: 10));
        actions.Add(new FleeFromThreatAction(cost: 15));
        actions.Add(new WaitAction(cost: 1));

        // Profile 特化
        if (profile == AiProfileKind.Hunter)
        {
            actions.Add(new BiteEnemyAction(cost: 8)); // 低 cost = 更倾向攻击
        }
        else
        {
            actions.Add(new BiteEnemyAction(cost: 30)); // 高 cost = 不倾向攻击
        }

        return actions.ToArray();
    }
}
```

这样不需要修改框架，也能先验证 3 个 Profile 的行为差异。等 Profile 行为真的成立，再把通用能力沉淀回框架。

### 缺口 C+K：游戏层 Decision Trace

不等框架补 PlannerTrace。游戏层自建 `CreatureAiDecisionTrace`：

```csharp
public sealed class CreatureAiDecisionTrace
{
    public RuntimeFrame Frame;
    public Dictionary<string, bool> FactsSnapshot;
    public string SelectedGoal;
    public string SelectedAction;
    public int ActionCost;
    public string Reason;
}
```

在 `CreatureAiRuntimeModule` 每次规划后记录一条 trace。游戏层收集 trace 序列，格式化为 Mission Report。

### 缺口 E+F：游戏层 HazardVolume

不等框架补。游戏层自建最小版：

```csharp
public sealed class GameHazardVolume
{
    public int VolumeId;
    public Aabb Bounds;
    public int DamagePerTick;
    public int BuffId; // 0 = 不挂 Buff
}
```

在 `EcoMapRuntimeModule` 每 tick 查询 `CombatPhysicsWorld` 区域内实体，施加伤害/Buff。

### 缺口 H：MVP 用手工 Waypoint

不等框架补 NavGraph。MVP 用：

```
固定 Waypoint 数组
手工连边
BFS 求路径
无动态障碍
无地形 cost
无 portal 限制
```

等多房间和生态地图再加 A*、cost 和 portal 限制。

---

# 4. 框架复用与自建决策

| 游戏需求 | 决策 | 理由 |
|----------|------|------|
| 主循环 | **复用** RuntimeHost + CommandBuffer | 框架核心能力，直接用 |
| 属性系统 | **复用** AttributeStore | v1 稳定，直接用 |
| Buff 系统 | **复用** BuffPipeline | v1 稳定，直接用 |
| AI 规划 | **复用** SequentialPlanner | v1 可用，Profile 用游戏层 Adapter |
| 碰撞查询 | **复用** CombatPhysicsWorld | v1 稳定，直接用 |
| 移动 | **复用** CombatKinematicMotor | v0.1 可用 |
| 实体管理 | **复用** GameplayComponentWorld | v0.1 可用 |
| 配置 | **复用** ConfigTable + ConfigSchema | v1 稳定，直接用 |
| 回放/存档 | **复用** Replay + SaveState | v0.1 可用 |
| Debug UI | **复用** DebugSourceRegistry + Timeline | v0.1-v0.2 可用 |
| 任务系统 | **自建** MissionRuntimeModule | 框架无通用 Mission 系统 |
| 导航 | **自建** 手工 Waypoint + BFS | 框架无寻路，MVP 不需要 A* |
| 危险区 | **自建** GameHazardVolume | 框架无持续区域伤害 |
| 生态模拟 | **自建** EcoMapRuntimeModule | 纯游戏层逻辑 |
| 战后报告 | **自建** Diagnostics 模块 | 游戏层格式化，复用框架 trace |
| 角色控制 | **包 adapter** CreatureControlAdapter | 复用 CharacterControl 思路 |

---

# 5. 游戏层模块划分

```
Assets/Scripts/VVV/
├── Creature/              # 生物定义、身体部件、实例创建
├── CreatureAi/            # AI Profile Adapter、Fact/Goal/Action、Decision Trace
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
2. AiProfile Adapter 生成 goals/actions（游戏层硬编码）
3. SequentialPlanner 规划 → 输出 plan
4. 游戏层记录 DecisionTrace
5. 取第一个 action → 转成 CreatureCommand
6. Movement / Combat / Mission 执行
7. 写事件、hash、trace
8. Debug UI 读取快照
```

关键简化：

- AI 只每隔 N 帧决策一次（避免抖动）
- Action 执行中不每帧重规划
- 只执行 plan 的第一个 action

---

# 7. 确定性回放的代价

M1-M2 强调 Replay hash 一致。这对确定性是正确的，但要注意成本：

1. **浮点运算**：框架已用 Fix64/FixVector3。游戏层新增的任何运算也必须遵守。
2. **每加一个系统都要重新验证 hash**：M4 引入战斗、M5 引入部件属性、M7 引入 HazardVolume — 每次合并 PR 都要跑 Replay hash 回归测试。
3. **小概率事件**：必须用 deterministic RNG seed。
4. **物理引擎**：不能用 Unity Physics 做权威，框架已处理（CombatPhysicsWorld）。

**必须尽早自动化 Replay hash CI**，而不是手工跑。否则到 M7 时会发现某个浮点累加在不同机器上差 1 位，然后回溯一个月的提交。
