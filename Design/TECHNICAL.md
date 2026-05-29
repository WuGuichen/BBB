# TECHNICAL — 技术方案

> 版本 3.1 | 2026-05-28

---

# 0. Stimulus–Belief–Utility Bus（最先实现）

> 所有实体实现 `IEmitStimulus + IReadStimulus`。总线是整个系统的脊椎。
> 详细规范见 [System_Core.md](System_Core.md)。

## 0.1 接口定义

```csharp
public interface IEmitStimulus
{
    EmitterDef[] GetEmitters();  // 设计师填的名词
}

public interface IReadStimulus
{
    SensorDef[] GetSensors();    // 设计师填的名词
    void OnStimulusReceived(Stimulus stim);  // 形成 Belief
}
```

## 0.2 总线运行时

```csharp
public sealed class StimulusBus
{
    // 每帧：收集所有 IEmitStimulus 发射的 Stimulus
    // 传播：按通道规则（视觉直线/嗅觉扩散/听觉球面）
    // 分发：所有 IReadStimulus 在范围内的接收
    // 形成 Belief → 进入 Utility 计算
}
```

## 0.3 战斗结算 — 无暗骰子

```csharp
// 伤害 = 攻击方 DamageAttribute × 防御方 ArmorAttribute
// 没有随机浮动。结果 100% 可追溯到玩家本可知道的原因。
public int CalculateDamage(int attack, int defense)
{
    return Mathf.Max(0, attack - defense);
}
```

## 0.4 模块目录（总线相关）

```
Assets/Scripts/WGame/
├── Bus/                   # Stimulus-Belief-Utility Bus
│   ├── Stimulus/          # Stimulus 结构体、通道/标签枚举
│   ├── Emitter/           # IEmitStimulus、EmitterDef
│   ├── Sensor/            # IReadStimulus、SensorDef
│   ├── Propagation/       # 传播规则（每通道一套）
│   ├── Belief/            # Belief 生命周期（形成/衰减/过期）
│   └── Utility/           # 效用计算、犹豫检测
```

---

# 1. 框架映射

| 框架系统 | 游戏层用途 |
|----------|-----------|
| Runtime Foundation | 主循环、Replay/Hash、SaveState |
| Attributes | 生物属性（HP/攻击/移速/SensorRange） |
| Buffs | 持续效果（恐惧/兴奋/出血） |
| Modifiers | 部件加成 |
| Config | 部件/地图/传感器配置 |
| Runtime AI Planner | 生物AI行为规划（Goal/Action/Utility） |
| Combat Physics | 碰撞、接触伤害、传感器范围查询 |
| Combat Motion | 移动、AABB阻挡 |
| Gameplay Component | 实体生命周期 |
| Debug UI | 调试快照 + Decision Trace Overlay |

---

# 2. 感知系统

## 2.1 视觉

```
CombatPhysicsWorld.Query(Sphere) → 命中实体 → 揭开迷雾
范围 = SensorRange.Vision 属性值
```

## 2.2 嗅觉

```
CombatPhysicsWorld.Query(Sphere) → 命中标记实体 → 生成气味流
方向有±30°误差，距离只有远/中/近三档
```

## 2.3 迷雾

视觉层实现。不进权威状态。

---

# 3. 身体部件系统

```csharp
public sealed class PartDefinition
{
    public int PartId;
    public string Name;
    public PartCategory Category;   // Core/Locomotion/Sensor/Weapon/Defense/Utility/Brain/Social
    public PartRarity Rarity;
    public float Weight;
    public float EnergyCost;
    public int[] AttributeModifierIds;
    public int[] AiModuleIds;
}
```

## 3.1 预算系统

```csharp
public sealed class BuildBudget
{
    public float WeightLimit;
    public float EnergyLimit;
    public int BrainCapacity;
}
```

---

# 4. AI蓝图系统

## 4.1 蓝图结构

```csharp
public sealed class AiBlueprint
{
    public string BlueprintId;
    public List<AiRule> Rules;
    public Dictionary<string, float> BlackboardVars;
    public StateMachine States;
}

public sealed class AiRule
{
    public string RuleId;
    public List<AiCondition> Conditions;
    public AiIntent Intent;
    public float Weight;
    public float Cooldown;
}
```

## 4.2 Utility打分

```csharp
foreach (var rule in blueprint.Rules)
{
    if (rule.Conditions.All(c => c.Evaluate(blackboard)))
    {
        intentScores[rule.Intent] += rule.Weight;
    }
}
```

---

# 5. 内部需求系统

```csharp
public sealed class NeedState
{
    public Fix64 Hunger;      // 0-100
    public Fix64 Fatigue;     // 0-100
    public Fix64 Fear;        // 0-100
    public Fix64 Curiosity;   // 0-100
    public Fix64 Aggression;  // 0-100
    public Fix64 Homesick;    // 0-100
}
```

---

# 6. 信号系统

```csharp
public sealed class Signal
{
    public string SignalId;
    public SignalType Type;
    public AiGoalPriorityShift[] GoalShifts;
    public float PersonalityResistance;
}
```

---

# 7. Run管理系统

## 7.1 Run状态

```csharp
public sealed class RunState
{
    public int CurrentZone;           // 当前生态区索引
    public int TotalZones;            // 总生态区数
    public List<CreatureEntity> Party; // 当前生物
    public RunInventory Inventory;     // Run内部件/信号/食物
    public SampleInventory Samples;    // 样本（死亡保留）
    public DataInventory Data;         // 数据（死亡保留）
    public bool IsRetreating;          // 是否正在撤退
}
```

## 7.2 Run内资源 vs Meta资源

```csharp
// Run内部件：装在生物身上或在背包里，死了丢
public class RunInventory
{
    public List<PartInstance> Parts;
    public List<Signal> Signals;
    public List<Food> Food;
}

// 样本：完成任务目标获得，死了保留
public class SampleInventory
{
    public Dictionary<int, int> Samples; // sampleId -> count
}

// 数据：观察行为/失败分析获得，死了保留
public class DataInventory
{
    public Dictionary<int, int> Data; // dataType -> amount
}
```

## 7.3 撤退机制

```csharp
public class RetreatSystem
{
    // 玩家选择撤退
    public void Retreat(RunState run)
    {
        // 带回所有Run内资源
        MetaProgress.AddParts(run.Inventory.Parts);
        MetaProgress.AddSamples(run.Samples);
        MetaProgress.AddData(run.Data);
        // Run结束
        run.End();
    }
    
    // 全灭
    public void Wipe(RunState run)
    {
        // 丢失Run内资源
        // 保留样本和数据
        MetaProgress.AddSamples(run.Samples);
        MetaProgress.AddData(run.Data);
        // Run结束
        run.End();
    }
}
```

## 7.4 Meta进度

```csharp
public class MetaProgress
{
    public HashSet<int> UnlockedParts;      // 解锁的部件选项
    public HashSet<int> UnlockedAiRules;    // 解锁的AI规则
    public HashSet<int> UnlockedZones;      // 解锁的生态区
    public int HatcherySlots;               // 孵化槽位
    public int StationLevel;                // 实验站等级
}
```

---

# 8. 主循环

```
RuntimeHost Tick
    ├── RunStateModule              (Run管理：当前区/撤退/全灭)
    ├── MapRevealModule             (迷雾)
    ├── WorldSimulationModule       (多智能体仿真)
    │   ├── AgentAiModule           (每个活物的AI Planner)
    │   ├── AgentNeedModule         (需求系统)
    │   └── EnvironmentModule       (环境规则)
    ├── LootModule                  (部件/信号/食物掉落)
    ├── InventoryModule             (Run内背包)
    ├── CreatureSensorModule        (感知→Belief，受 PartRuntimeState 修正)
    ├── CreatureMemoryModule        (短期记忆)
    ├── CreatureAiModule            (玩家生物AI，Utility 受部件损伤修正)
    ├── CreatureBodyModule          (部件状态/损伤/输出能力) ← 进hash，权威
    ├── CreatureMotionModule        (移动，受 MoveSpeed 修正)
    ├── CreatureContactResolver     (接触结算：attack-defense，无暗骰)
    ├── CreatureCombatModule        (战斗后果→新 Stimulus 注入总线)
    ├── SignalModule                (信号系统)
    ├── ProceduralBodyViewModule    (程序化身体表现：摆动/拖行/断裂) ← 不进hash
    ├── HighlightModule             (AI反馈四层)
    ├── DecisionTraceModule         (开发期调试)
    └── Diagnostics / Replay / Hash
```

**CreatureBodyModule（进 hash）**：维护每个实体的 `PartRuntimeState[]`。部件损伤修改 SensorDef/EmitterDef/Attribute，直接影响 Belief 形成和 Utility 计算。

**ProceduralBodyViewModule（不进 hash）**：读取 CreatureBodyModule 的权威状态，翻译为程序化身体表现：
- 部件完好 → 正常摆动/朝向
- 部件 Weakened → 拖行/减速动画
- 部件 Disabled → 挂在身上不动
- 部件 Detached → 飞出
- 部件 Leaking → 粒子从身体漏出

**接触结算（CreatureContactResolver）**：`damage = attack - defense`，无随机浮动。结果 100% 可追溯到部件属性。

---

# 9. 游戏层模块

```
Assets/Scripts/WGame/
├── Body/                  # 身体部件系统
│   ├── Parts/
│   ├── Budget/
│   └── Assembly/
├── Ai/                    # AI蓝图系统
│   ├── Blueprint/
│   ├── Rules/
│   ├── Conditions/
│   ├── Intents/
│   └── Utility/
├── World/                 # 多智能体仿真
│   ├── Agents/
│   ├── Needs/
│   ├── Environment/
│   ├── Ecology/
│   └── Persistence/
├── Sensory/               # 感知系统
├── Signal/                # 信号系统
├── Run/                   # Run管理
│   ├── RunState/
│   ├── RunInventory/
│   ├── Retreat/
│   └── ZoneTransition/
├── Meta/                  # Meta进度
│   ├── Unlocks/
│   ├── Station/
│   └── Samples/
├── Task/                  # 任务系统
├── Report/                # 战后报告
├── Map/                   # 房间模板
├── Camera/                # 2.5D等距相机
├── DebugSandbox/          # 调试场景
├── Demo/
└── Tests/
```

---

# 10. Decision Trace Overlay

每次AI决策记录：

```
Frame 1832
Known Facts: enemy.visible, hunger.high, fear.medium
Top Goals: Attack=72, Flee=68, Eat=44
Decision: Hesitate (delta=4 < threshold=15)
```

---

# 11. 视觉与相机规格

**2.5D等距俯视**。相机角度55-60°。

跟随滞后：200-300ms。

关键时刻zoom：
- 犹豫：1.0→1.1
- 战斗接触：1.0→1.15，慢放0.5s

---

# 12. 确定性

**全栈确定性。** 不用 Unity 内置 PhysX。

```
数值层：Fix64 / FixVector3 / FixMatrix
物理层：定点数 AABB 碰撞检测 + 固定步长物理循环
逻辑层：所有 Gameplay 模块（Utility/Combat/Body/Signal）
随机性：集中在 RunRandom seed
渲染层：读取权威状态，不参与计算
```

## 12.1 物理系统

**不用 Unity PhysX。** 自研定点数物理：

- 碰撞体：AABB / OBB / 球（定点数）
- 碰撞检测：GJK/SAT（定点数运算）
- 动力学：固定步长（不依赖 DeltaTime）
- solver：单线程，确定性迭代顺序
- 碰撞回调：确定性顺序

**Replay 100% 一致。** 不同平台、不同硬件、不同帧率 → 同样的结果。

## 12.2 Hash 分层

| 层 | 进 hash | 说明 |
|----|---------|------|
| Gameplay 逻辑 | ✅ | Utility/Combat/Body/Signal |
| 物理状态 | ✅ | 位置/速度/碰撞结果 |
| 部件状态 | ✅ | PartRuntimeState |
| 渲染表现 | ❌ | 程序化身体/粒子/相机 |
| UI | ❌ | 头顶图标/行为日志 |
