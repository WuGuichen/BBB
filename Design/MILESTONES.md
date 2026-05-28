# MILESTONES — 制作计划

> 版本 0.5 | 2026-05-27
>
> 本文描述开发阶段、验收条件和风险。技术映射见 [TECHNICAL.md](TECHNICAL.md)，玩法设计见 [GDD.md](GDD.md)。

---

# 1. 阶段总览

| 阶段 | 目标 | 核心验证 |
|------|------|----------|
| **M0** | Behavior Lab 骨架 | RuntimeHost 能 Tick，占位生物可见 |
| **M1** | 歧义地图 + 确定性移动 | 生物在三路地图中移动不穿墙，Replay hash 一致 |
| **M2** | 行为旋钮 + AI 基础 | 旋钮改变路线选择，Memory 影响行为 |
| **M3** | Quick Test | 旋钮调整 30 秒内看到行为差异 |
| **M4** | 信号 cooldown | 玩家通过信号影响 AI 决策，犹豫提示可见 |
| **M5** | 任务闭环 + Fast-forward | 完整回收任务，1x/2x/4x/8x，Retry Same + Variant Retry |
| **M6** | Attempt Diff + 涌现高亮 | 玩家能从报告理解"改了什么导致了什么" |
| **M7** | 身体部件 v0 | 部件影响任务结果，和旋钮/地图有交互 |
| **M8** | 基础战斗 | 接触威胁 + Bite + HP + Death |
| **M9** | Profile v0 | TaskFirst/Hunter/Coward 作为旋钮预设 |

**关键原则**：先验证"AI 行为调教是否好玩"，再加部件、战斗、生态。

---

# 2. 各阶段详情

## M0：Behavior Lab 骨架

**目标**：VVV 游戏层成立，RuntimeHost 正常 Tick。

要做：
1. 新建 `Assets/Scripts/VVV/` 游戏层目录
2. 新建 `VVVRuntimeCompositionRoot`
3. 接入 `RuntimeHost` + `RuntimeCommandBuffer`
4. 建立最小 Debug HUD（UI Toolkit）

验收：
- Play 后有占位生物
- Frame 增长
- Reset 后状态一致

---

## M1：歧义地图 + 确定性移动

**目标**：Lab Field 01 三路地图可玩，单体生物移动不穿墙。

复用：`CombatPhysicsWorld` + `CombatKinematicMotor`

要做：
1. `EcoMapDefinition` + `EcoRoomDefinition`
2. Lab Field 01：三岔路（短路/长路/侧路）+ AABB 障碍
3. 资源点 + 出口 + 敌人巡逻点
4. `CreatureMotionAdapter`
5. 视觉同步（占位几何体 + 路线颜色区分）

验收：
- 生物能走到三条路中的任意一条
- 撞墙不穿透
- Replay hash 一致
- 不同帧率结果一致

---

## M2：行为旋钮 + AI 基础

**目标**：旋钮改变 AI 行为，Memory 让 AI 有"过去"。

复用：`Runtime AI Planner` + `CreatureAiProfileRuntimeAdapter`

要做：
1. 5 个行为旋钮（Aggression/Caution/Curiosity/Greed/HomeBias）
2. 旋钮→Goal priority / Action cost 映射
3. 旋钮张力关系 UI 提示
4. `CreatureFactKeys`（resource.visible / enemy.visible / threat.near / hp.low / path.shortRisky / path.longSafe / time.low）
5. 3 个 Memory（lastThreatPosition / lastResourcePosition / avoidZoneMemory）
6. Route choice 作为 Intent（TakeShortRoute / TakeSafeRoute）
7. `CreatureAiDecisionTrace`

验收：
- Aggression 高时 AI 更倾向走短路
- Caution 高时 AI 更倾向走长路
- Memory：AI 闻到威胁后即使敌人不在视野仍绕路
- 同一输入 Replay hash 100% 一致
- **同一地图至少 3 种旋钮配置可成功完成任务**

---

## M3：Quick Test

**目标**：旋钮调整 30 秒内看到行为差异。

要做：
1. Quick Test 模式（3 个 10 秒场景：遇敌/见资源/危险区）
2. 结果摘要（攻击/绕行/逃跑 × 拾取/忽略 × 穿越/绕路）
3. Quick Test 入口 UI

验收：
- 旋钮调整后 30 秒内看到行为差异
- Quick Test 结果和完整任务中的行为一致
- **主观 Playtest**：3 个非开发者能通过 Quick Test 理解旋钮效果

---

## M4：信号 cooldown

**目标**：玩家通过信号影响 AI 决策，任务中有干预点。

要做：
1. Avoid Signal（标记区域为危险，AvoidThreat +30，持续 8 秒）
2. Recall Signal（ReturnToExit +80，持续 15 秒）
3. Signal Charge（每局 3 次，cooldown 10 秒）
4. 犹豫提示"？"（最高分和第二分差距小时显示）
5. 信号使用 UI

验收：
- Avoid Signal 能让 AI 改走安全路线
- Recall Signal 能让 AI 中途返回
- 犹豫提示在 AI 不确定时可见
- 信号有 10 秒 cooldown，不能连续使用
- **主观 Playtest**：3 个非开发者能说出"我在关键时刻发了信号"

---

## M5：任务闭环 + Fast-forward

**目标**：完整回收任务可玩，支持加速和重试。

要做：
1. `MissionRuntimeModule`（CollectResource + ReturnToExit）
2. Fast-forward（1x/2x/4x/8x）
3. Retry Same Seed
4. Variant Retry（改变敌人位置/资源位置/危险区状态）
5. 成功/失败判定

验收：
- 完整任务可跑通（拿资源 → 返回出口）
- 4x/8x 不影响结果（确定性）
- Retry Same Seed 结果一致
- Variant Retry 产生不同结果
- **决策密度**：3 分钟内至少 2 个有效信号使用点
- **主观 Playtest**：3 个非开发者能复述"这只生物刚才为什么这么做"

---

## M6：Attempt Diff + 涌现高亮

**目标**：玩家能从报告理解"改了什么导致了什么"，AI 聪明行为被看见。

要做：
1. Causal Summary（3-5 个关键节点的因果链，自然语言）
2. Attempt Diff（和上次同地图对比：旋钮变化→行为变化→结果变化）
3. 涌现高亮（AI 因 Memory/Signal 选择非默认路线时主动提示）
4. 实验评分标签（险胜/高效回收/过度谨慎/贪婪致死/聪明绕路）
5. **Data 奖励去重**（新失败模式给大量 Data，重复失败递减）

验收：
- 因果摘要不超过 5 个节点，自然语言可读
- Attempt Diff 能指出"smell_sensor 导致绕路"
- 涌现高亮在 AI 记住威胁绕路时触发
- **主观 Playtest**：3 个非开发者能根据 Attempt Diff 说出"下次我该调什么"
- **90% 失败能归因到 1 条主要因果链**

---

## M7：身体部件 v0

**目标**：部件影响任务结果，和旋钮/地图有交互。

复用：`AttributeStore.AddModifier` + `ConfigModifierFactory`

要做：
1. `BodyPartDefinitionConfig` — 6 类，每类 2 个
2. 部件→属性修改器 / Action 解锁 / Fact 解锁映射
3. **部件×环境交互**（fast_legs 穿短路快但转向差，smell_sensor 和 Caution 联动）

验收：
- fast_legs 平均完成时间比 stable_legs 短 ≥ 15%
- armor_shell 生存率比 thin_skin 高 ≥ 20%
- smell_sensor 能在无视野时找到资源
- **反协同验证**：fast_legs + armor_shell 在预算约束下不能严格优于其他组合

---

## M8：基础战斗

**目标**：接触威胁给任务制造压力。

要做：
1. 接触伤害（敌人碰到玩家 → 受伤 → 可逃跑）
2. Bite（Sphere query + DamageByAttackDefense）
3. 敌人 AI（巡逻 + 看到玩家后追击）
4. 死亡 → 尸体状态
5. 威胁估计（threat.stronger based on enemy stats）

验收：
- 弱敌可被击杀
- 强敌可杀死玩家生物
- 死亡原因进入因果报告
- 战斗事件进入涌现高亮（如"它打了两口发现打不过，带伤撤退了"）

---

## M9：Profile v0

**目标**：Profile 作为旋钮预设，降低新手门槛。

要做：
1. TaskFirst / Hunter / Coward 三个预设
2. 每个预设 = 旋钮值 + 推荐部件
3. 选择预设后自动填充旋钮

验收：
- 三个预设的行为差异可感知
- Brain View 能解释差异
- 玩家可以从预设出发微调旋钮

---

# 3. 最小竖切内容清单

| 类别 | 内容 |
|------|------|
| **地图** | Lab Field 01：三岔路 + 资源 + 出口 + 敌人 + 危险区 |
| **生物** | Player Creature：HP / MoveSpeed / Attack / Defense |
| **敌人** | Dummy Beast：巡逻 + 看到玩家后追击 |
| **旋钮** | Aggression / Caution / Curiosity / Greed / HomeBias |
| **信号** | Avoid Signal / Recall Signal（各 3 次/局，cooldown 10 秒） |
| **Memory** | lastThreatPosition / lastResourcePosition / avoidZoneMemory |
| **UI** | 旋钮面板 / Quick Test / 信号按钮 / 犹豫提示 / Fast-forward / Retry |
| **报告** | Causal Summary / Attempt Diff / 涌现高亮 / 实验评分 |

---

# 4. Playtest 节点

| 阶段 | Playtest 内容 |
|------|--------------|
| M3 | 3 个非开发者通过 Quick Test 理解旋钮效果 |
| M5 | 3 个非开发者完成一次任务，能复述 AI 行为原因 |
| M6 | 3 个非开发者能根据 Attempt Diff 说出"下次我该调什么" |
| M7 | 3 个非开发者能感受到部件差异 |
| M9 | 3 个非开发者能区分三个 Profile 的行为风格 |

---

# 5. 风险清单

| 风险 | 应对 |
|------|------|
| 涌现 vs 噪声 | Brain View 自然语言归因 + 涌现高亮 + M3/M5/M6 Playtest |
| 任务中只能看戏 | 信号 cooldown + 犹豫提示 + 决策密度验收 |
| 设计粒度太粗 | 行为旋钮（5 个带张力）代替纯 Profile |
| 反馈循环太慢 | Quick Test（30 秒）+ Fast-forward + Retry Same Seed |
| 构筑有最优解通杀 | 预算约束 + 部件×环境交互 + 歧义地图 |
| AI 黑箱 | 因果报告 + Attempt Diff + Memory 事件可追溯 |
| 认知负担太高 | 预设 Profile → Quick Test → Attempt Diff 三层降低 |
| 最强构筑通杀 | M7 反协同验证 |
| 确定性回放 hash 回归 | 尽早自动化 Replay hash CI |

---

# 6. 暂缓清单

| 功能 | 原因 |
|------|------|
| Decision Checkpoint（强制暂停） | 节奏风险，用犹豫提示+信号 cooldown 代替 |
| AI Confidence / Commitment | 解释负担高，等 playtest 验证后再加 |
| 自由 AI 卡片编辑器 | 先用旋钮，后期再开放 |
| 完整战斗系统 | 先做接触威胁，验证行为深度后再加 |
| 生态模拟 / 多物种 | 先验证单体行为调教 |
| 程序化身体建模 | 玩法未验证 |
| 繁殖 / 多人 / Mod | 不在 MVP |
| 正式美术 | 全用占位体 |

---

# 7. 框架调整 Issue 建议（暂不创建）

待游戏层验证成立后，在 WGameFramework 仓库创建：

| 优先级 | Issue | 内容 |
|--------|-------|------|
| P0 | [AI] Goal Priority Modifier | 给 PriorityGoalSelector 增加 IGoalPriorityModifier 接口 |
| P0 | [AI] Action Cost Modifier | 给 IAiAction 增加 cost modifier 机制 |
| P0 | [AI] PlannerTrace Output | 给 SequentialPlanner 增加 trace 输出 + DebugUI 集成 |
| P1 | [Combat] HazardVolume | 新增 HazardVolume 概念 + tick 伤害/Buff 系统 |
| P1 | [Navigation] Minimal NavGraph | 纯 C# A* + Waypoint/Edge + portal 限制 |
| P2 | [Gameplay] Inventory Component | 通用 Inventory 组件 + hash + SaveState |
