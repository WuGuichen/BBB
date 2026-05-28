# TECHNICAL — 技术方案与框架映射

> 版本 0.7 | 2026-05-27

---

# 1. 框架能力总览

| 框架系统 | 成熟度 | 游戏层用途 |
|----------|--------|-----------|
| **Runtime Foundation** | ✅ v0.1 | 主循环、命令缓冲、Replay/Hash、SaveState |
| **Attributes** | ✅ v1 | 生物属性（HP/攻击/移速） |
| **Buffs** | ✅ v1 | 持续效果 |
| **Modifiers** | ✅ v1 | 零件加成 |
| **Config** | ✅ v1 | 零件/地图/节点配置 |
| **Runtime AI Planner** | ✅ v1 | 生物 AI 行为规划 |
| **Combat Physics** | ✅ v1 | 碰撞查询、接触伤害 |
| **Combat Motion** | ✅ v0.1 | 固定帧移动、AABB 阻挡 |
| **Gameplay Component** | ✅ v0.1 | 实体生命周期 |
| **Debug UI** | ✅ v0.1-v0.2 | 调试快照 |
| **Input** | ✅ v0.1 | 信号/侦察/撤退操作 |

---

# 2. 系统映射

## 2.1 零件 → Attributes + Modifiers + AI 模块

```csharp
public sealed class PartDefinition
{
    public int PartId;
    public PartRarity Rarity;      // White / Blue / Purple
    public PartType Type;          // Body / AiModule / Signal
    public int InventorySlots;     // 背包占格
    public int[] AttributeModifierIds;
    public int[] AiModuleIds;
    public string[] UnlockFacts;
}
```

## 2.2 AI 模块 → Runtime AI Planner

AI 模块改变 Goal priority 和 Action cost。模块可矛盾叠加。

```csharp
public sealed class AiModuleDefinition
{
    public int ModuleId;
    public AiGoalPriorityShift[] GoalShifts;
    public AiActionCostShift[] ActionShifts;
}
```

矛盾叠加示例：

```
勇气核心：AttackThreat +60, Flee cost +30
谨慎核心：AvoidThreat +60, Attack cost +40
都装 → AttackThreat +60, AvoidThreat +60
→ 高血量时 Attack 胜出，低血量时 Avoid 胜出
→ 中间血量产生犹豫（差距小）
```

## 2.3 小队 → 多实体

MVP 2 个小队槽位，每只是独立实体：

```csharp
public sealed class SquadSlot
{
    public CreatureInstance Creature;  // 独立实体
    public SquadPosition Position;     // Front / Back
}
```

每只生物独立 AI 决策，独立移动，独立战斗。小队没有共享状态机 — 性格冲突从独立决策中涌现。

## 2.4 信号 → 临时 AiWorldState 修改

信号是一次性道具，使用时对指定生物临时修改 Utility 分数。

## 2.5 背包 → 自建 Inventory

```csharp
public sealed class PlayerInventory
{
    public int MaxSlots;           // 8
    public List<InventoryEntry> Items;
    public List<PartDefinition> CoreParts;  // 核心零件，死亡不丢
}
```

## 2.6 星图 → 自建 RunGraph

```csharp
public sealed class RunGraph
{
    public RunNode[] Nodes;        // 5-7 per layer
    public RunEdge[] Edges;        // 分支连接
    public int Layers;             // 3
}
```

节点类型：Normal / Elite / Event / Rest

## 2.7 战争迷雾 → 视觉层

Unity 视觉层实现，不进权威状态。

## 2.8 战斗 → 接触伤害 + 可观察性

MVP：接触伤害 + 0.3 秒前摇 + 音效 + 伤害数字弹出。

```csharp
public sealed class ContactDamageModule
{
    // 生物 overlap 敌人 → 触发前摇 → 结算伤害 → 弹出数字
    public int PreHitFrames;  // 18帧 ≈ 0.3秒
}
```

## 2.9 主循环 → RuntimeHost

```
RuntimeHost Tick
    ├── RunStateModule              (Run 管理、星图、节点)
    ├── MapRevealModule             (战争迷雾)
    ├── LootModule                  (零件掉落)
    ├── InventoryModule             (背包)
    ├── SquadModule                 (小队管理)
    ├── CreatureSensorModule × N    (每只生物独立传感器)
    ├── CreatureAiModule × N        (每只生物独立 AI)
    ├── CreatureMotionModule × N    (每只生物独立移动)
    ├── CreatureCombatModule × N    (每只生物独立战斗)
    ├── SignalModule                (信号使用)
    ├── HighlightModule             (性格高亮 + 犹豫提示)
    └── Diagnostics / Replay / Hash
```

---

# 3. 框架缺口

| # | 缺口 | 策略 |
|---|------|------|
| A/B | AI Planner 无 Goal/Action modifier | 游戏层 AiModule 硬编码偏移 |
| C+K | 无 PlannerTrace | 游戏层自建 DecisionTrace |
| J | 无 Inventory | 游戏层自建 |

MVP 不需要 HazardVolume / NavGraph。

---

# 4. 游戏层模块

```
Assets/Scripts/VVV/
├── Run/                   # Run 管理、星图、节点、死亡/保留
├── Map/                   # 地图、碰撞、迷雾、宝箱
├── Loot/                  # 零件掉落、零件池
├── Inventory/             # 背包、核心零件
├── Squad/                 # 小队管理、队形
├── Assembly/              # 组装（部件+模块→生物）
├── Creature/              # 生物、属性
├── CreatureAi/            # AI 模块映射、DecisionTrace
├── CreatureSensor/        # 传感器、迷雾揭示
├── CreatureCombat/        # 接触伤害、前摇、伤害数字
├── Signal/                # 信号道具
├── Highlight/             # 性格高亮、犹豫提示
├── Demo/                  # 场景入口
└── Tests/                 # 测试
```

---

# 5. 确定性

Fix64/FixVector3。Replay hash CI 自动化。
