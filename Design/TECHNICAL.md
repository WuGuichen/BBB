# TECHNICAL — 技术方案

> 版本 3.0 | 2026-05-28

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
    ├── CreatureSensorModule        (感知→AiWorldState)
    ├── CreatureMemoryModule        (短期记忆)
    ├── CreatureAiModule            (玩家生物AI)
    ├── CreatureMotionModule        (移动)
    ├── CreatureCombatModule        (接触伤害)
    ├── SignalModule                (信号系统)
    ├── HighlightModule             (AI反馈四层)
    ├── DecisionTraceModule         (开发期调试)
    └── Diagnostics / Replay / Hash
```

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

- Fix64 / FixVector3
- Replay hash CI
- 视觉层不进hash
- 随机性集中在RunRandom seed
