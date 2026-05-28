# TECHNICAL — 技术方案

> 版本 0.8 | 2026-05-27

---

# 1. 框架映射

| 框架系统 | 游戏层用途 |
|----------|-----------|
| Runtime Foundation | 主循环、Replay/Hash、SaveState |
| Attributes | 生物属性（HP/攻击/移速） |
| Buffs | 持续效果 |
| Modifiers | 零件加成 |
| Config | 零件/地图/传感器配置 |
| Runtime AI Planner | 生物 AI 行为规划 |
| Combat Physics | 碰撞、接触伤害、传感器范围查询 |
| Combat Motion | 移动、AABB 阻挡 |
| Gameplay Component | 实体生命周期 |
| Debug UI | 调试快照 |

---

# 2. 感知系统

## 2.1 视觉

```
CombatPhysicsWorld.Query(Sphere) → 命中实体 → 揭开迷雾
范围 = SensorRange 属性值
黑暗区范围缩小
```

## 2.2 嗅觉

```
CombatPhysicsWorld.Query(Sphere) → 命中标记实体 → 生成气味流
不显示精确位置，只显示方向和强度
AI 产生 threat.smell fact（confidence 0.5-0.7）
```

## 2.3 迷雾

视觉层实现。Unity shader 或 tilemap。不进权威状态。

---

# 3. AI 系统

## 3.1 AI 模块 → Runtime AI Planner

模块改变 Goal priority / Action cost。可矛盾叠加。

## 3.2 短期记忆

```csharp
public sealed class CreatureMemory
{
    public FixVector2? LastThreatPosition;   // 最近被攻击的方向
    public FixVector2? LastResourcePosition; // 最近发现资源的位置
    public int TrustLevel;                   // 对玩家信号的信任度
}
```

Memory 写入 AiWorldState facts。

## 3.3 信任度

```csharp
// 玩家信号帮助了生物 → TrustLevel += 20
// 玩家信号坑了生物 → TrustLevel -= 30
// TrustLevel 低时信号效果减弱或无效
```

Post-MVP。MVP 先硬编码信任度不存在。

---

# 4. 干预系统

信号是道具。使用时临时修改指定生物的 AiWorldState。

```csharp
public sealed class InterventionVerb
{
    public string VerbId;           // "warn" / "suggest" / "calm" / "encourage"
    public AiGoalPriorityShift[] GoalShifts;
    public int DurationFrames;
    public float PersonalityResistance; // 不同性格的抵抗系数
}
```

勇气核心对"安抚"的 resistance 高。谨慎核心对"鼓舞"的 resistance 高。

---

# 5. 零件系统

```csharp
public sealed class PartDefinition
{
    public int PartId;
    public PartRarity Rarity;
    public PartType Type;        // Body / AiModule / Signal
    public int InventorySlots;
    public int[] AttributeModifierIds;
    public int[] AiModuleIds;
}
```

---

# 6. 主循环

```
RuntimeHost Tick
    ├── RunStateModule              (Run 管理)
    ├── MapRevealModule             (迷雾)
    ├── LootModule                  (零件掉落)
    ├── InventoryModule             (背包)
    ├── CreatureSensorModule        (视觉/嗅觉 → AiWorldState)
    ├── CreatureMemoryModule        (短期记忆)
    ├── CreatureAiModule            (AI 规划)
    ├── CreatureMotionModule        (移动)
    ├── CreatureCombatModule        (接触伤害，3-5秒)
    ├── InterventionModule          (干预动词)
    ├── HighlightModule             (性格高亮 + 犹豫提示)
    └── Diagnostics / Replay / Hash
```

---

# 7. 游戏层模块

```
Assets/Scripts/VVV/
├── Sensory/               # 感知系统（视觉/嗅觉/迷雾）
├── Creature/              # 生物、属性、记忆
├── CreatureAi/            # AI 模块映射、DecisionTrace
├── CreatureCombat/        # 接触伤害
├── Intervention/          # 干预动词
├── Highlight/             # 性格高亮、犹豫提示
├── Run/                   # Run 管理
├── Map/                   # 地图、宝箱
├── Loot/                  # 零件掉落
├── Inventory/             # 背包
├── Demo/                  # 场景入口
└── Tests/                 # 测试
```

---

# 8. 确定性

Fix64/FixVector3。Replay hash CI。
