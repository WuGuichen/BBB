# TECHNICAL — 技术方案

> 版本 2.0 | 2026-05-28
>
> 冲突时本文档高于 Sensory_Design / GDD / CONCEPT。

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
    public PartRarity Rarity;       // White/Blue/Purple
    public float Weight;
    public float EnergyCost;
    public float Noise;
    public float Durability;
    public int[] AttributeModifierIds;
    public int[] AiModuleIds;
    public bool IsCorePart;         // 死亡不丢
}
```

## 3.1 预算系统

```csharp
public sealed class BuildBudget
{
    public float WeightLimit;       // 体重预算
    public float EnergyLimit;       // 能量预算
    public int BrainCapacity;       // 脑容量（可装AI规则数）
    public float MutationStability; // 突变稳定度
}
```

---

# 4. AI蓝图系统

## 4.1 蓝图结构

```csharp
public sealed class AiBlueprint
{
    public string BlueprintId;
    public List<AiRule> Rules;          // 规则卡列表
    public Dictionary<string, float> BlackboardVars;
    public StateMachine States;
}

public sealed class AiRule
{
    public string RuleId;
    public List<AiCondition> Conditions; // 条件卡
    public AiIntent Intent;              // 意图卡
    public float Weight;                 // 权重
    public float Cooldown;
}
```

## 4.2 Utility打分

```csharp
// 每帧计算每个Intent的分数
foreach (var rule in blueprint.Rules)
{
    if (rule.Conditions.All(c => c.Evaluate(blackboard)))
    {
        intentScores[rule.Intent] += rule.Weight;
    }
}
// 选择最高分的Intent执行
```

## 4.3 玩家配置界面

玩家不写代码，用卡片式界面：
- 选择条件卡（看到敌人/血量低/饥饿高...）
- 选择意图卡（攻击/逃跑/调查/进食...）
- 调权重
- 排列优先级

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

需求影响Utility打分：

```csharp
// 饥饿>70时，进食Intent权重+50
if (needs.Hunger > 70) intentScores[Eat] += 50;
// 恐惧>60时，逃跑Intent权重+40
if (needs.Fear > 60) intentScores[Flee] += 40;
```

---

# 6. 信号系统

```csharp
public sealed class Signal
{
    public string SignalId;
    public SignalType Type;         // Beacon/Warning/Recall/Calm/Stimulate
    public AiGoalPriorityShift[] GoalShifts;
    public float PersonalityResistance; // 不同AI蓝图的抵抗系数
}
```

信号不直接控制行为，而是改变Utility分数。AI设计太极端，生物可能不听。

---

# 7. 主循环

```
RuntimeHost Tick
    ├── RunStateModule              (Run管理)
    ├── MapRevealModule             (迷雾)
    ├── WorldSimulationModule       (多智能体仿真)
    │   ├── AgentAiModule           (每个活物的AI Planner)
    │   ├── AgentNeedModule         (需求系统)
    │   └── EnvironmentModule       (环境规则)
    ├── LootModule                  (零件掉落)
    ├── InventoryModule             (背包)
    ├── CreatureSensorModule        (感知→AiWorldState)
    ├── CreatureMemoryModule        (短期记忆)
    ├── CreatureAiModule            (玩家生物AI，含需求交互)
    ├── CreatureMotionModule        (移动)
    ├── CreatureCombatModule        (接触伤害)
    ├── SignalModule                (信号系统)
    ├── HighlightModule             (AI反馈四层)
    ├── DecisionTraceModule         (开发期调试)
    └── Diagnostics / Replay / Hash
```

---

# 8. 游戏层模块

```
Assets/Scripts/WGame/
├── Body/                  # 身体部件系统
│   ├── Parts/             # 部件定义
│   ├── Budget/            # 预算系统
│   └── Assembly/          # 组装逻辑
├── Ai/                    # AI蓝图系统
│   ├── Blueprint/         # 蓝图定义
│   ├── Rules/             # 规则卡
│   ├── Conditions/        # 条件卡
│   ├── Intents/           # 意图卡
│   └── Utility/           # Utility打分
├── World/                 # 多智能体仿真
│   ├── Agents/            # 独立AI主体
│   ├── Needs/             # 需求系统
│   ├── Environment/       # 环境规则
│   ├── Ecology/           # 生态位
│   └── Persistence/       # 跨Run持久化
├── Sensory/               # 感知系统
├── Signal/                # 信号系统
├── Task/                  # 任务系统
├── Station/               # 实验站
├── Report/                # 战后报告
├── Map/                   # 房间模板、活物配置
├── Camera/                # 2.5D等距相机
├── DebugSandbox/          # 调试场景
├── Demo/                  # 场景入口
└── Tests/                 # 测试
```

---

# 9. Decision Trace Overlay

开发期必备调试系统。

每次AI决策记录：
```
Frame 1832
Known Facts: enemy.visible, hunger.high, fear.medium
Top Goals: Attack=72, Flee=68, Eat=44
Decision: Hesitate (delta=4 < threshold=15)
```

UI形态：Unity Game View叠加层（F1切换），支持时间回溯。

---

# 10. 视觉与相机规格

**2.5D等距俯视**。相机角度55-60°。

跟随滞后：200-300ms。

关键时刻zoom：
- 犹豫：1.0→1.1
- 战斗接触：1.0→1.15，慢放0.5s
- 拾取：微推0.05

---

# 11. 确定性

- Fix64 / FixVector3
- Replay hash CI
- 视觉层不进hash
- 随机性集中在RunRandom seed

---

# 12. 测试

## 12.1 单元

- 部件属性计算正确性
- AI Blueprint规则评估正确性
- 需求系统状态变化正确性
- 信号GoalShift正确性

## 12.2 集成

- Replay hash跨帧率一致
- Decision Trace输出与AI Planner实际选择一致

## 12.3 Playtest

- 同一房间3次播放事件不同
- 玩家能说出"我下次想试试不同的设计"
