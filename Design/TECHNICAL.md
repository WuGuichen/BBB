# TECHNICAL — 技术方案

> 版本 0.9 | 2026-05-27
>
> 冲突时本文档高于 Sensory_Design / GDD / CONCEPT。

---

# 1. 框架映射

| 框架系统 | 游戏层用途 |
|----------|-----------|
| Runtime Foundation | 主循环、Replay/Hash、SaveState |
| Attributes | 生物属性（HP/攻击/移速/SensorRange） |
| Buffs | 持续效果（恐惧/兴奋/出血） |
| Modifiers | 零件加成 |
| Config | 零件/地图/传感器配置 |
| Runtime AI Planner | 生物 AI 行为规划（Goal/Action/Utility） |
| Combat Physics | 碰撞、接触伤害、传感器范围查询 |
| Combat Motion | 移动、AABB 阻挡 |
| Gameplay Component | 实体生命周期 |
| Debug UI | 调试快照 + Decision Trace Overlay |

---

# 2. 感知系统

## 2.1 视觉

```
CombatPhysicsWorld.Query(Sphere) → 命中实体 → 揭开迷雾
范围 = SensorRange.Vision 属性值
黑暗区范围缩小（环境变量 Modifier）
```

输出：实体在生物认知层标记为 `enemy.visible` / `resource.visible` / `exit.visible`，confidence 1.0。

## 2.2 嗅觉

```
CombatPhysicsWorld.Query(Sphere) → 命中标记实体 → 生成气味流
不显示精确位置，只显示方向（±30° 误差）和强度（远/中/近 3 档）
AI 产生 threat.smell / resource.smell fact（confidence 0.5-0.7）
```

视觉表现层：粒子系统沿"生物 → 气味源"方向生成连续飘动粒子，颜色由 fact 类型决定。

## 2.3 迷雾

视觉层实现。Unity shader 或 tilemap mask。**不进权威状态**（不参与 hash）。

记忆衰减规则：
- 当前可见区域：`fog_alpha = 0`
- 曾经可见但当前不在视野：`fog_alpha = 0.4`，动态实体位置**冻结在最后可见的位置**
- 完全未知：`fog_alpha = 1.0`

动态实体冻结实现：渲染层维护"最后可见位置缓存"，离开视野时锁定，重新进入视野时刷新。

---

# 3. AI 系统

## 3.1 AI 模块 → Runtime AI Planner

模块改变 Goal priority / Action cost。可矛盾叠加。

```csharp
public sealed class AiModuleDefinition
{
    public int ModuleId;
    public AiGoalPriorityShift[] GoalShifts;  // 例: {Attack +40}
    public AiActionCostShift[] ActionShifts;
    public PersonalitySignature Signature;     // 性格三表现规格的数据载体
}
```

矛盾叠加：勇气 GoalShift[Attack +40] 与 谨慎 GoalShift[Flee +35] 同时存在，Utility 在 HP 维度上分段产生不同主导。

## 3.2 短期记忆

```csharp
public sealed class CreatureMemory
{
    public FixVector2? LastThreatPosition;       // 最近被攻击的方向
    public List<FixVector2> KnownResourcePositions;
    public List<FixVector2> KnownDangerZones;    // 进入此区域时朝向锁定
    public Fix64 TrustLevel;                     // 0.0-1.0，Trust v0/v1 共用
    public int RecentDamageTakenInFrames;
}
```

Memory 每帧写入 AiWorldState facts（`memory.last_threat_direction`, `player.trust` 等）。

## 3.3 信任度（双层）

### Trust v0 表现层 `MVP M3`

```csharp
// 内部变量驱动三处表现，不影响 AI 决策
public Fix64 TrustLevel;  // 0.0-1.0，初始 0.5

// 规则
// 玩家信号后生物存活 30s → TrustLevel += 0.05
// 玩家信号后生物受重伤 30s → TrustLevel -= 0.10
// 玩家忽略生物的犹豫 → TrustLevel -= 0.02

// 表现钩子
// 1. 犹豫时刻头顶图标变体
// 2. 信号反馈短语模板选择
// 3. Run 结束总结一句变体
```

### Trust v1 系统层 `Post-MVP M6`

```csharp
// 信任度进入 fact，影响 AI 接受信号的强度
AiWorldState.SetFact("player.trust", TrustLevel);

// InterventionVerb 的 GoalShift 倍率 = base * f(TrustLevel)
//   f(1.0) = 1.5
//   f(0.5) = 1.0
//   f(0.2) = 0.5
//   f(0.0) = 0   // 完全忽略
```

**v0 → v1 升级路径**：v0 的内部变量直接作为 v1 的 fact 值。表现层规则不变，只追加决策层接入点。

---

# 4. 干预系统

信号是道具（M7 之后）或固定供给（M4-M6）。使用时临时修改指定生物的 AiWorldState。

```csharp
public sealed class InterventionVerb
{
    public string VerbId;                  // "warn" / "suggest" / "calm" / "encourage"
    public AiGoalPriorityShift[] GoalShifts;
    public int DurationFrames;
    public float PersonalityResistance;    // 不同性格的抵抗系数
    public float TrustMultiplier;          // 由 Trust v1 决定（v0 阶段固定 1.0）
}
```

## 4.1 反馈结果分类（结果层数据源）

```csharp
public enum InterventionOutcome
{
    ObeyedAndSurvived,   // 听了 + 30s 内生存
    ObeyedAndWounded,    // 听了 + 30s 内受重伤
    Refused,             // 没听（PersonalityResistance 拦截）
    Misled,              // 听了但导致伤亡（玩家坑了它）
}
```

每次干预后，HighlightModule 根据 outcome 选择对应反馈短语；Trust 模块根据 outcome 调整 TrustLevel。

---

# 5. 零件系统

```csharp
public sealed class PartDefinition
{
    public int PartId;
    public PartRarity Rarity;              // White / Blue / Purple
    public PartType Type;                  // Body / AiModule / Signal
    public int InventorySlots;
    public int[] AttributeModifierIds;
    public int[] AiModuleIds;
    public bool IsCorePart;                // 死亡不丢
}
```

---

# 6. 主循环（RuntimeHost Tick）

```
RuntimeHost Tick
    ├── RunStateModule              (Run 管理)
    ├── MapRevealModule             (迷雾，视觉层不入 hash)
    ├── LootModule                  (零件掉落)
    ├── InventoryModule             (背包)
    ├── CreatureSensorModule        (视觉/嗅觉 → AiWorldState)
    ├── CreatureMemoryModule        (短期记忆 + Trust v0/v1)
    ├── CreatureAiModule            (AI 规划)
    ├── CreatureMotionModule        (移动)
    ├── CreatureCombatModule        (接触伤害，3-5秒)
    ├── InterventionModule          (干预动词 + Outcome 回写)
    ├── HighlightModule             (性格高亮 + 犹豫提示 + 结果层短语)
    ├── DecisionTraceModule         (开发期，见 §9)
    └── Diagnostics / Replay / Hash
```

---

# 7. 游戏层模块

```
Assets/Scripts/VVV/
├── Sensory/               # 感知系统（视觉/嗅觉/迷雾/记忆衰减）
├── Creature/              # 生物、属性、记忆、命名、伤痕
├── CreatureAi/            # AI 模块、Personality Signature、Decision Trace
├── CreatureCombat/        # 接触伤害
├── Intervention/          # 干预动词 + Outcome 分类
├── Trust/                 # Trust v0/v1
├── Highlight/             # 性格高亮、犹豫提示、结果层短语
├── Run/                   # Run 管理、Mission Report
├── Map/                   # 房间模板、宝箱、节拍元素
├── Loot/                  # 零件掉落
├── Inventory/             # 背包
├── Camera/                # 2.5D 等距相机（见 §11）
├── DebugSandbox/          # Sensory Sandbox 调试场景（见 §10）
├── Demo/                  # 场景入口
└── Tests/                 # 测试
```

---

# 8. 确定性

- Fix64 / FixVector3
- Replay hash CI（每次 PR 必跑）
- 视觉层（迷雾/粒子/相机）不进 hash
- 随机性集中在 `RunRandom` seed，可重放

---

# 9. Decision Trace Overlay

> **开发期必备调试系统**。玩家不可见。决定"AI 不可读"是 UI / 数值 / 节奏 / 地图哪一层问题的关键工具。

## 9.1 每次 AI 决策记录

```
Frame 1832
Known Facts:
- enemy.visible       confidence 1.0
- threat.smell        confidence 0.6   direction 45° distance:near
- hp.low              false
- resource.visible    true
- memory.last_threat  east

Top Goals:
1. AttackEnemy        = 72   勇气核心 +40, enemy.visible +30, memory.last_threat +2
2. AvoidThreat        = 68   谨慎核心 +35, threat.smell +20, memory.last_threat +13
3. PickResource       = 44   贪欲核心 +30, resource.visible +14

Decision:
- Hesitate            top1-top2 delta = 4 (threshold 15)
- Player-facing:      "它在犹豫：勇气想打，谨慎想躲"
```

## 9.2 UI 形态

- Unity Game View 叠加层（F1 切换）
- 显示当前焦点生物的最新一次决策
- 支持时间回溯（左右箭头逐帧）
- 数据落盘到 `Trace_{frame}_{creatureId}.json`，方便复现分析

## 9.3 用途

- 判断 Playtest 玩家"不懂 AI"的归因：
  - 决策合理但日志不清晰 → 理由层措辞问题
  - 决策合理但表现层不可读 → 行为/情绪层问题
  - 决策本身不合理 → 数值或 fact 问题
  - 节拍密度低 → 地图问题

---

# 10. Sensory Sandbox 调试场景

> **M0 配套**。所有 Playtest 在这个场景里跑，不用启动完整 Run 流程。

## 10.1 功能

```
- 固定房间布局（Sensory §7 验收剧本的房间）
- 一键重置（R 键）
- 热切换 AI 模块（数字键 1-9）
- 热切换身体部件
- 无限信号道具（Tab 切换信号种类）
- 时间控制（空格暂停 / 1234 慢放速率）
- Decision Trace Overlay 默认开启
```

## 10.2 不属于 Sandbox 的部分

- 不掉零件
- 不掉血流失
- 不算 Run 进度
- 不写 SaveState（独立 session）

## 10.3 用途

- Playtest 默认场景
- 单元测试用例驱动场景
- 美术/动画/特效迭代

---

# 11. 视觉与相机规格

> v0.9 新增。锁定视角方向，避免 M1 时相机漂移导致重做。

## 11.1 视角

**2.5D 等距俯视**。相机角度 55-60°（比标准 45° 等距更俯视，便于读地图）。

- 投影：正交（MVP）或弱透视（Post-MVP）
- 单位标准高度 1m
- 单位标准占地 0.5m × 0.5m

## 11.2 相机行为

```
默认：跟随焦点生物
跟随滞后：200-300ms（让生物的转身/停顿在画面产生视觉惯性）

事件触发：
- 犹豫时刻：zoom 1.0 → 1.1，时长 0.3s
- 战斗接触：zoom 1.0 → 1.15，慢放 0.5s
- 拾取：zoom 微推 0.05，时长 0.3s

玩家控制：
- Tab：切换焦点生物（MVP 单生物，预留接口）
- 右键拖动：自由滑动（仅限已揭开区域）
- 空格：半暂停（时间 0.25x，相机自由）
```

## 11.3 占位体规格

| 元素 | 占位形态 | 表现关键点 |
|------|---------|----------|
| 生物 | 圆柱体 1m × 0.5m + 朝向箭头 + 状态图标头顶悬浮 | 朝向必须明显，6 种状态颜色区分 |
| 敌人 | 圆柱体红色边框 | 与生物可区分 |
| 资源 | 立方体 + 金色发光 | 远距离可见 |
| 宝箱 | 立方体 + 闪烁 | 拾取后变灰 |
| 墙 | 灰色方块 | 高度 2m，遮挡视线 |
| 危险地形 | 半透明红色覆盖 | 不阻挡移动但触发伤害 |

**MVP 阶段必须做到**：占位体看 5 秒能区分种类、朝向、状态。**做不到就改占位规格，不是等美术**。

## 11.4 感知视觉特效

| 元素 | 实现 |
|------|------|
| 视觉揭开 | Shader mask，圆形 + 软边 |
| 嗅觉气流 | 粒子系统，沿生物→源方向飘动，颜色由 fact 决定 |
| 听觉涟漪 | Post-MVP，同心圆 shader |
| 已揭开记忆 | 颜色 desaturate 60% + 动态实体冻结位置 |
| 头顶图标 | UI Canvas world-space，紧跟生物 |
| 犹豫光晕 | 生物脚下圆形脉冲 |

---

# 12. 测试

## 12.1 单元

- Sensor 范围查询正确性
- AI Planner Goal 选择正确性（给定 facts → 期望 goal）
- 信号 GoalShift 正确性
- Trust v0 变量驱动短语模板选择
- 记忆衰减规则

## 12.2 集成

- Replay hash 跨帧率一致
- Sensory Sandbox 一键重置后状态完全一致
- Decision Trace 输出与 AI Planner 实际选择一致

## 12.3 Playtest 自动化

- 录制 Playtest 玩家操作 + 屏幕
- Decision Trace 自动落盘
- 配合诊断题问卷（MILESTONES §4）做归因分析
