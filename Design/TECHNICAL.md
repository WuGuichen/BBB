# TECHNICAL — 技术方案与框架映射

> 版本 0.6 | 2026-05-27

---

# 1. 框架能力总览

| 框架系统 | 成熟度 | 游戏层用途 |
|----------|--------|-----------|
| **Runtime Foundation** | ✅ v0.1 | 主循环、命令缓冲、Replay/Hash、SaveState |
| **Attributes** | ✅ v1 | 生物属性（HP/攻击/移速） |
| **Buffs** | ✅ v1 | 持续效果（中毒/加速） |
| **Modifiers** | ✅ v1 | 零件加成 |
| **Config** | ✅ v1 | 零件/地图/区域配置 |
| **Runtime AI Planner** | ✅ v1 | 生物 AI 行为规划 |
| **Combat Physics** | ✅ v1 | 碰撞查询、接触伤害 |
| **Combat Motion** | ✅ v0.1 | 固定帧移动、AABB 阻挡 |
| **Gameplay Component** | ✅ v0.1 | 实体生命周期 |
| **Debug UI** | ✅ v0.1-v0.2 | 调试快照 |
| **Input** | ✅ v0.1 | 信号使用、菜单操作 |

---

# 2. 系统映射

## 2.1 零件 → Attributes + Modifiers + Config

每个零件是配置驱动的修改器 + AI 模块绑定：

```
白色快腿：MoveSpeed +20%（AttributeModifier Mul）
蓝色快腿：MoveSpeed +35%, Noise +50%（两个 Modifier）
勇气核心：AttackThreat goal priority +60（AI 模块）
嗅觉腔：Unlock fact threat.smell（Sensor 解锁）
```

技术实现：

```csharp
public sealed class PartDefinition
{
    public int PartId;
    public string StableId;
    public PartRarity Rarity;      // White / Blue / Purple
    public PartType Type;          // Body / AiModule / Signal
    public int InventorySlots;     // 占背包几格
    public int[] AttributeModifierIds;  // 属性修改器
    public int[] AiModuleIds;          // AI 模块 ID
    public string[] UnlockFacts;       // 解锁的 Fact
}
```

## 2.2 AI 模块 → Runtime AI Planner

AI 模块改变 Goal priority 和 Action cost。一个生物装 1-2 个模块，效果叠加。

```csharp
public sealed class AiModuleDefinition
{
    public int ModuleId;
    public string StableId;
    public AiGoalPriorityShift[] GoalShifts;   // goal priority 偏移
    public AiActionCostShift[] ActionShifts;   // action cost 偏移
}
```

示例：

```
勇气核心：
  AttackThreat priority += 60
  Flee cost += 30

谨慎核心：
  AvoidThreat priority += 60
  Attack cost += 40

贪欲核心：
  CollectResource priority += 60
  ReturnToExit cost += 20
```

## 2.3 信号 → 临时 AiWorldState 修改

信号是一次性道具，使用时临时修改 AiWorldState：

```csharp
public sealed class SignalItemDefinition
{
    public int ItemId;
    public string StableId;
    public SignalEffect[] Effects;  // 临时 goal/fact 修改
    public int DurationFrames;      // 持续时间
}
```

召回信标：ReturnToExit priority +80，持续 15 秒。

## 2.4 背包 → 自建 Inventory

框架无 Inventory 组件。游戏层自建：

```csharp
public sealed class PlayerInventory
{
    public int MaxSlots;           // 8
    public List<InventoryEntry> Items;
    public bool TryAdd(PartDefinition part);  // 检查容量
    public void Remove(int index);
}
```

接入 RuntimeHost 的 SaveState，支持 Run 间存档。

## 2.5 战争迷雾 → 视觉层

MVP 的战争迷雾不需要框架支持。用 Unity 视觉层实现：

- 地图默认黑色
- 生物当前位置周围揭示一小片
- 已揭示区域保持可见

权威状态不需要迷雾 — 迷雾纯视觉。

## 2.6 自动战斗 → 接触伤害

MVP 战斗最简化：

```
生物 Sphere overlap 敌人 → 生物受伤
生物和敌人都有 HP
HP <= 0 → 死亡
```

不需要 Bite/Charge/Sector。接触就扣血，结果由 HP/Attack/Defense 决定。

## 2.7 地图 → CombatPhysics + 手工设计

MVP 地图手工设计（非程序化生成）：

```
AABB 墙壁/障碍
Sphere 资源点/宝箱/出口
敌人巡逻路径（固定 Waypoint）
危险区（AABB trigger zone）
```

## 2.8 主循环 → RuntimeHost

```
Unity Input / UI
    ↓
RuntimeCommandBuffer
    ↓
RuntimeHost Tick
    ├── RunStateModule              (Roguelike Run 管理)
    ├── MapRevealModule             (战争迷雾)
    ├── LootModule                  (零件掉落)
    ├── InventoryModule             (背包管理)
    ├── CreatureAssemblyModule      (组装生物)
    ├── CreatureSensorModule        (传感器)
    ├── CreatureAiModule            (AI 规划)
    ├── CreatureMotionModule        (移动，复用 CombatKinematicMotor)
    ├── CreatureCombatModule        (接触伤害)
    ├── SignalModule                (信号使用)
    ├── HighlightModule             (性格行为高亮 + 犹豫提示)
    └── Diagnostics / Replay / Hash
```

---

# 3. 框架缺口

| # | 缺口 | 策略 |
|---|------|------|
| A/B | AI Planner 无 Goal/Action modifier | 游戏层 AiModule 硬编码 goal/action 偏移 |
| C+K | 无 PlannerTrace | 游戏层自建 DecisionTrace |
| J | 无 Inventory | 游戏层自建 PlayerInventory |

**其余缺口（HazardVolume / NavGraph）在 MVP 不需要。** 接触伤害和手工地图够用。

---

# 4. 游戏层模块划分

```
Assets/Scripts/VVV/
├── Run/                   # Roguelike Run 管理、区域选择、死亡/保留
├── Map/                   # 地图定义、碰撞体、迷雾、资源点、宝箱
├── Loot/                  # 零件掉落、零件池、稀有度
├── Inventory/             # 背包管理
├── Assembly/              # 生物组装（部件+AI模块→生物实例）
├── Creature/              # 生物定义、属性、实例
├── CreatureAi/            # AI 模块映射、DecisionTrace
├── CreatureSensor/        # 传感器、战争迷雾揭示
├── CreatureCombat/        # 接触伤害、HP、死亡
├── Signal/                # 信号道具、使用效果
├── Highlight/             # 性格行为高亮、犹豫提示
├── Demo/                  # 场景入口
└── Tests/                 # 测试
```

---

# 5. 确定性

框架已用 Fix64/FixVector3。游戏层新增运算遵守。Replay hash CI 自动化。
