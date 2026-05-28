# TECHNICAL — 技术方案与框架映射

> 版本 0.5 | 2026-05-27
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
| **Modifiers** | ✅ v1 | 部件加成、旋钮修正 |
| **Config** | ✅ v1 | 部件/旋钮/任务/地图配置 |
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
| **Character Action** | 🟡 代码完成 | 动作系统（未接入运行管线） |

---

# 2. 系统映射

## 2.1 行为旋钮 → Runtime AI Planner

映射规则：每个旋钮值（0-100）映射到 Goal priority 偏移和 Action cost 偏移。

```csharp
public sealed class CreatureBehaviorKnobs
{
    public int Aggression;  // 0-100
    public int Caution;     // 0-100
    public int Curiosity;   // 0-100
    public int Greed;       // 0-100
    public int HomeBias;    // 0-100
}
```

映射表：

```
Aggression:
  AttackThreat goal priority += Aggression * 0.6
  Bite/Charge action cost -= Aggression * 0.3

Caution:
  AvoidThreat goal priority += Caution * 0.5
  Hazard path cost += Caution * 0.4
  Flee action cost -= Caution * 0.3

Curiosity:
  Investigate goal priority += Curiosity * 0.5
  Unknown stimulus attract += Curiosity * 0.3

Greed:
  OptionalResource goal priority += Greed * 0.6
  Return threshold delayed += Greed * 0.3

HomeBias:
  ReturnToExit goal priority += HomeBias * 0.5
  DistanceFromExit penalty += HomeBias * 0.4
```

**旋钮张力**（同时高时的行为变化）：

```
Aggression 高 + Caution 高：
  Attack 和 Avoid 都高 → AI 在两者间摇摆 → 产生"试探型"行为

Greed 高 + HomeBias 高：
  Collect 和 Return 都高 → AI 捡到东西就想回家 → 适合安全路线

Curiosity 高 + Caution 高：
  Investigate 和 Avoid 都高 → AI 调查但不敢靠近 → 侦察型
```

## 2.2 AI Memory → AiWorldState

Memory 通过写入 AiWorldState facts 实现：

```csharp
public sealed class CreatureMemory
{
    // 传感器发现威胁时写入
    public FixVector2? LastThreatPosition;
    // 传感器发现资源时写入
    public FixVector2? LastResourcePosition;
    // 玩家发 Avoid Signal 或进入危险区时写入
    public List<FixVector2> AvoidZoneMemory;
}
```

运行时流程：

```
Sensor Phase  → 收集世界信息 + 读取 Memory → 写入 AiWorldState facts
Policy Phase  → 旋钮调整 goal priority / action cost
Plan Phase    → SequentialPlanner.TryPlan
Command Phase → 取第一个 action
Trace Phase   → 记录 DecisionTrace（含 Memory 读写事件）
```

## 2.3 信号 → AiWorldState

信号通过临时修改 AiWorldState facts 实现：

```csharp
public sealed class SignalEffect
{
    public string FactKey;      // e.g. "signal.avoid_zone"
    public bool Value;
    public int DurationFrames;  // 持续时间
    public int CooldownFrames;  // 冷却时间
}
```

Avoid Signal：
- 写入 `signal.avoid_zone = true`（持续 8 秒）
- 对应 Goal：AvoidThreat +30
- 对应 Action：该区域 path risk +30

Recall Signal：
- 写入 `signal.recall = true`（持续 15 秒）
- 对应 Goal：ReturnToExit +80
- 对应 Fact：HomeBias 临时 +40

## 2.4 战斗 → Combat Physics

| 阶段 | 实现 |
|------|------|
| MVP 接触威胁 | 敌人 Sphere overlap 玩家 → 受伤 → 可逃跑 |
| Post-MVP Bite | `CombatPhysicsWorld.Query(Sphere)` + `DamageByAttackDefense` |
| Post-MVP Charge | `CombatKinematicMotor.MoveForward` + `Query(Capsule sweep)` |

## 2.5 地图 → Combat Physics

| 地图元素 | 框架实现 |
|----------|----------|
| 墙壁/障碍 | `CombatPhysicsWorld` AABB body + collider |
| 资源点 | Game layer component（Sphere trigger zone） |
| 出口 | Game layer component（Sphere trigger zone） |
| 危险区 | Game layer GameHazardVolume（AABB + damagePerTick） |
| 寻路 | MVP: 手工 Waypoint + BFS |

## 2.6 移动 → Combat Kinematic Motor

复用 `CombatKinematicMotor`。不同生物通过 Motor 参数区分：MaxSpeed、Acceleration、TurnRate。

## 2.7 主循环 → Runtime Host

```
Unity Input / UI
    ↓
RuntimeCommandBuffer
    ↓
RuntimeHost Tick
    ├── MissionRuntimeModule          (游戏层)
    ├── EcoMapRuntimeModule            (游戏层)
    ├── CreatureSensorModule           (游戏层)
    ├── CreatureMemoryModule           (游戏层)
    ├── CreatureKnobModule             (游戏层)
    ├── CreatureSignalModule           (游戏层)
    ├── CreatureAiRuntimeModule        (游戏层)
    ├── CreatureCommandModule          (游戏层)
    ├── CreatureMotionModule           (复用 CombatKinematicMotor)
    ├── CreatureCombatModule           (游戏层 + CombatPhysicsWorld)
    ├── Diagnostics / Replay / Hash    (复用框架)
    └── EmergentHighlightModule        (游戏层)
```

## 2.8 配置 → Config System

| 配置类型 | 对应 |
|----------|------|
| BodyPartConfig | 部件属性、解锁 Action、解锁 Fact、碰撞代理 |
| BehaviorKnobConfig | 旋钮→Goal/Action 映射参数 |
| MissionConfig | 地图 ID、目标列表、Signal Charge 数量 |
| EcoMapConfig | Room 定义、Hazard、Resource、Waypoint |

---

# 3. 框架缺口分析

## 3.1 缺口总表

| # | 系统 | 缺口 | 优先级 | 策略 |
|---|------|------|--------|------|
| A | AI Planner | Goal Priority 无动态调整 | **P0** | 游戏层硬编码旋钮映射 |
| B | AI Planner | Action Cost 无 modifier 机制 | **P0** | 游戏层硬编码旋钮映射 |
| C+K | AI Planner + Debug | 无 PlannerTrace 输出 | **P0** | 游戏层自建 DecisionTrace |
| E+F | Combat | 无 HazardVolume | **P1** | 游戏层自建 |
| H | Navigation | 无 NavGraph / A* | **P1** | MVP 用手工 Waypoint+BFS |
| J | Gameplay | 无 Inventory 组件 | **P2** | 游戏层自建 |

## 3.2 核心策略：游戏层先硬编码验证

**当功能只服务 WGame_B 时，优先游戏层实现。当两个以上项目需要时，再上升为框架能力。**

具体做法：

### 缺口 A/B：旋钮映射 Adapter

```csharp
public sealed class CreatureAiProfileRuntimeAdapter
{
    public IAiGoal[] CreateGoals(CreatureBehaviorKnobs knobs, MemoryState memory)
    {
        var goals = new List<IAiGoal>();
        goals.Add(new FactGoal("collect.resource",
            priority: 50 + (int)(knobs.Greed * 0.6)));
        goals.Add(new FactGoal("return.exit",
            priority: 40 + (int)(knobs.HomeBias * 0.5)));
        goals.Add(new FactGoal("avoid.death",
            priority: 30 + (int)(knobs.Caution * 0.5)));
        goals.Add(new FactGoal("attack.threat",
            priority: 20 + (int)(knobs.Aggression * 0.6)));
        goals.Add(new FactGoal("investigate",
            priority: 10 + (int)(knobs.Curiosity * 0.5)));
        return goals.ToArray();
    }
}
```

### 缺口 C+K：游戏层 DecisionTrace

```csharp
public sealed class CreatureAiDecisionTrace
{
    public RuntimeFrame Frame;
    public Dictionary<string, object> FactsSnapshot;
    public string SelectedGoal;
    public string SelectedAction;
    public int ActionCost;
    public string Reason;
    public bool IsEmergentMoment; // 是否是涌现时刻
    public string EmergentExplanation; // 涌现解释
}
```

### 缺口 E+F：游戏层 HazardVolume

```csharp
public sealed class GameHazardVolume
{
    public int VolumeId;
    public Aabb Bounds;
    public int DamagePerTick;
    public int BuffId;
}
```

---

# 4. 框架复用与自建决策

| 游戏需求 | 决策 | 理由 |
|----------|------|------|
| 主循环 | **复用** RuntimeHost + CommandBuffer | 框架核心 |
| 属性/Buff/Modifier | **复用** AttributeStore / BuffPipeline / ModifierPipeline | v1 稳定 |
| AI 规划 | **复用** SequentialPlanner | v1 可用，旋钮用游戏层 Adapter |
| 碰撞查询 | **复用** CombatPhysicsWorld | v1 稳定 |
| 移动 | **复用** CombatKinematicMotor | v0.1 可用 |
| 实体/配置/回放 | **复用** 框架对应模块 | 稳定可用 |
| 任务系统 | **自建** MissionRuntimeModule | 框架无通用 Mission |
| 信号系统 | **自建** CreatureSignalModule | 纯游戏层逻辑 |
| Memory 系统 | **自建** CreatureMemoryModule | 纯游戏层逻辑 |
| 旋钮系统 | **自建** CreatureKnobModule | 纯游戏层逻辑 |
| 涌现高亮 | **自建** EmergentHighlightModule | 纯游戏层逻辑 |
| Quick Test | **自建** QuickTestModule | 纯游戏层逻辑 |
| Attempt Diff | **自建** AttemptDiffModule | 纯游戏层逻辑 |
| 导航 | **自建** 手工 Waypoint + BFS | 框架无寻路 |
| 危险区 | **自建** GameHazardVolume | 框架无持续区域伤害 |

---

# 5. 游戏层模块划分

```
Assets/Scripts/VVV/
├── Creature/              # 生物定义、身体部件、实例创建
├── CreatureAi/            # 旋钮映射、Fact/Goal/Action、DecisionTrace
├── CreatureMemory/        # Memory 系统（lastThreat/lastResource/avoidZone）
├── CreatureSignal/        # 信号系统（Avoid/Recall、cooldown、charge）
├── CreatureKnob/          # 行为旋钮（5 个旋钮、张力关系、预设）
├── CreatureCombat/        # 接触威胁、攻击判定、伤害结算
├── EcoMap/                # 房间图、碰撞体、资源点、出口、危险区、Waypoint
├── Mission/               # 任务目标、状态、成功/失败
├── Diagnostics/           # 因果报告、Attempt Diff、涌现高亮、实验评分
├── QuickTest/             # 30 秒快速测试模式
├── Demo/                  # 场景入口、组合根
└── Tests/                 # 单元测试、集成测试、PlayMode 测试
```

---

# 6. 运行时管线

单个生物每帧流程：

```
1. Sensor 收集世界信息 + 读取 Memory → 写入 AiWorldState facts
2. 旋钮映射 → 生成 goals + actions + costs
3. 信号检查 → 临时修改 facts/goals
4. SequentialPlanner 规划 → 输出 plan
5. 游戏层记录 DecisionTrace + 检测涌现时刻
6. 取第一个 action → 转成 CreatureCommand
7. Movement / Combat / Mission 执行
8. Memory 更新（如有新感知事件）
9. 写事件、hash、trace
10. Debug UI / Brain View / 涌现高亮 读取快照
```

---

# 7. 确定性回放的代价

M1-M2 强调 Replay hash 一致。成本：

1. **浮点运算**：框架已用 Fix64/FixVector3，游戏层新增运算也必须遵守
2. **每加一个系统都要重新验证 hash**：M2 旋钮、M4 信号、M7 部件 — 每次合并 PR 跑 Replay hash 回归
3. **小概率事件**：必须用 deterministic RNG seed
4. **物理引擎**：不能用 Unity Physics 做权威

**必须尽早自动化 Replay hash CI。**
