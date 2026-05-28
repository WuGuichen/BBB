# GDD — AI Creature Sandbox 玩法设计

> 版本 0.5 | 2026-05-27
>
> 本文描述"这个游戏怎么玩、为什么好玩"。技术映射见 [TECHNICAL.md](TECHNICAL.md)，制作计划见 [MILESTONES.md](MILESTONES.md)。
>
> 每个系统标注阶段：`MVP` / `Post-MVP` / `Long-term`

---

# 1. 玩家身份与世界观

玩家是**边境实验站管理员**。

- 通过回收基因、骨骼、机械部件，创造适应环境的半生物半机械造物
- 将造物投放到危险生态区执行任务
- 回收资源，解锁新部件和 AI 模块
- 弱化人类角色，视觉重点放在怪物、机械生物、低模生态、实验设施

---

# 2. 核心循环

## 行为实验循环 `MVP`

```
提出行为假设
↓
调身体与行为旋钮
↓
快速测试（30 秒 Quick Test）
↓
投放歧义地图
↓
用有限信号干预（Avoid / Recall，带 cooldown）
↓
捕捉涌现时刻（AI 聪明行为高亮）
↓
查看 Attempt Diff（和上次对比）
↓
微调并复测
```

**玩家不是看 AI 表演，玩家是在做行为实验。**

## 时间尺度

| 尺度 | 内容 |
|------|------|
| **30 秒微循环** | 生物发现了什么？为什么这么做？安全吗？要不要发信号？AI 是否误判？ |
| **3 分钟循环** | 一次任务：2-4 个信号决策 + 1 次涌现时刻 + 1 份 Attempt Diff |
| **30 分钟循环** | 完成多轮实验：找到可行构筑 + 理解 2-3 种失败模式 |

## 决策密度目标

3 分钟任务内：
- 任务前：3 个构筑决策（旋钮 + 部件）
- 任务中：2-4 个干预决策（信号）
- 任务后：1-2 个调整决策（微调旋钮）

---

# 3. 身体系统

## 3.1 八类部件 `MVP: 6类` `Post-MVP: +Utility` `Long-term: +Social`

| 类型 | 决定什么 | 示例 | 阶段 |
|------|----------|------|------|
| **Core 核心** | 基础形态、能量容量 | 小型核心（轻/脆）、重型核心（厚/慢） | `MVP` |
| **Locomotion 移动** | 移动方式 | 双足（均衡）、四足（稳定追猎）、六足（地形适应） | `MVP` |
| **Sensor 感知** | AI 能知道什么 | 普通眼、夜视眼、嗅觉腔（追踪） | `MVP` |
| **Weapon 攻击** | 攻击方式 | 爪、牙、角、毒刺 | `MVP` |
| **Defense 防御** | 生存能力 | 甲壳、鳞片、脂肪层、酸抗脚垫 | `MVP` |
| **Brain 脑模块** | AI 复杂度 | 原始脑（少量规则）、猎手脑（攻击加成）、战术脑 | `MVP` |
| **Utility 功能** | 特殊能力 | 携带囊、挖掘爪、隐身膜 | `Post-MVP` |
| **Social 社交** | 群体行为 | 信息素腺体、吼叫器官、信号天线 | `Long-term` |

## 3.2 部件属性 `MVP`

每个部件带标签：重量、能耗、噪音、耐久、维修难度、生物质消耗、金属消耗、恐惧影响、攻击倾向影响、适配地形、兼容部件、AI 需求。

## 3.3 预算约束 `MVP`

防止堆满最强部件：体重预算、能量预算、脑容量预算、突变稳定度、部件槽位、维护成本。

**预算/维护成本是底层数学，不是 Post-MVP 装饰。** MVP 就必须有反协同 — fast_legs + armor_shell 组合不能严格优于其他组合。

## 3.4 部件与环境交互 `MVP`

部件不只是数值调整，必须和地图产生交互：

```
Fast Legs：快速穿越短路，暴露时间降低，但遇敌后转向差
Smell Sensor：提前发现威胁，Caution 高时绕路，Curiosity 高时可能调查气味
Armor：允许硬吃短路敌人，但速度慢，长路可能超时
```

---

# 4. AI 行为系统

## 4.1 行为旋钮 `MVP`

MVP 不做自由规则编辑器，给 5 个行为旋钮：

| 旋钮 | 含义 | 映射 |
|------|------|------|
| **Aggression** | 攻击性 | AttackThreat +priority, Bite/Charge -cost |
| **Caution** | 谨慎度 | AvoidThreat +priority, Hazard path cost +, Flee -cost |
| **Curiosity** | 好奇心 | Investigate +priority, Unknown stimulus attract + |
| **Greed** | 贪婪度 | OptionalResource +priority, Return threshold delayed |
| **HomeBias** | 返巢倾向 | ReturnToExit +priority, DistanceFromExit penalty + |

Profile 是旋钮预设，玩家可以微调：

```
Hunter:  Aggression 高, Caution 低, Curiosity 中, Greed 低, HomeBias 低
Coward:  Aggression 低, Caution 高, Curiosity 低, Greed 低, HomeBias 高
TaskFirst: Aggression 低, Caution 中, Curiosity 低, Greed 中, HomeBias 高
```

**旋钮之间有张力**，UI 必须显示：

```
Aggression ↔ Caution：同时偏高会导致攻击和撤退之间反复试探
Greed ↔ HomeBias：同时偏高会边捡边想回家，适合安全路线不适合高危地图
Curiosity ↔ Caution：同时偏高会调查但不敢靠近，适合侦察但浪费时间
```

## 4.2 AI Memory `MVP: 3个` `Post-MVP: +`

让 AI 不只是响应当前帧，而是有"过去"：

| Memory | 写入条件 | 影响 |
|--------|----------|------|
| **lastThreatPosition** | 传感器发现威胁 | 即使威胁不在视野，仍绕开该区域 |
| **lastResourcePosition** | 传感器发现资源 | 记住资源位置，下次优先前往 |
| **avoidZoneMemory** | 玩家发 Avoid Signal 或进入危险区 | 持续避开该区域 |

Memory 写入事件必须进入因果报告，让玩家理解"它不是乱跑"。

## 4.3 基础状态 `MVP`

9 个状态：Idle / Explore / Forage / Combat / Flee / Recover / Return / Protect / Nest

## 4.4 意图 Intent `MVP`

```
CollectObjective / ReturnToExit / AvoidThreat / AttackThreat
InvestigateStimulus / TakeShortRoute / TakeSafeRoute
FollowSignal / Wait
```

注意：路线选择本身是意图，这增加决策深度。

## 4.5 预制 Profile `Post-MVP`

Post-MVP 给 3 个预制 Profile（TaskFirst / Hunter / Coward），每个是旋钮预设 + 部件推荐。

---

# 5. 玩家介入方式

## 5.1 任务前设计 `MVP`

在实验站配置：身体部件（3 个选择：Movement/Sensor/Defense）、行为旋钮（5 个）。

## 5.2 信号系统 `MVP`

不直接控制，用有限信号改变 AI Utility 分数：

| 信号 | 效果 | 持续 | 冷却 |
|------|------|------|------|
| **Avoid Signal** | 标记区域为危险，AvoidThreat +30, path risk +30 | 8 秒 | 10 秒 |
| **Recall Signal** | ReturnToExit +80, HomeBias 临时 +40 | 15 秒 | 10 秒 |

每局 3 次 Signal Charge。

**AI 可能不完全服从** — 狂暴型可能忽略 Recall，饥饿型可能忽略 Avoid。这让信号不是命令，而是影响倾向。

## 5.3 犹豫提示 `MVP`

AI 出现不确定性时（最高分和第二分差距小），头顶出现"？"图标，Brain View 高亮当前分歧，提示玩家"现在发信号效果最大"。不强制暂停。

## 5.4 任务后分析 `MVP`

参见 §7 反馈系统 — Attempt Diff、因果报告、涌现高亮。

---

# 6. 地图与任务

## 6.1 歧义地图 `MVP`

第一张地图不是测试房间，而是"最小策略场"：

### Lab Field 01：三路回收测试场

```
起点/出口
↓
三岔路
A. 短路：最快，但有敌人巡逻（高风险高收益）
B. 长路：安全，但可能超时（稳健型）
C. 侧路：有额外资源，但有危险区（贪婪型）
↓
任务资源
↓
返回出口
```

变量（支持 Variant Retry）：
- 敌人巡逻初始位置
- 可选资源位置
- 危险区开关

**同一张图至少存在 3 种可行策略**：高速冒险、稳健绕路、感知避敌。

## 6.2 生态地图类型 `Post-MVP`

| 地图 | 结构 | 核心测试 |
|------|------|----------|
| 开阔盆地 | 大平地，少量遮挡 | 速度，追击 |
| 窄洞网络 | 狭窄通道 | 感知，伏击 |
| 危险湿地区 | 酸池，毒气 | 抗性，路线规划 |

## 6.3 物种分布 `Post-MVP`

SpawnScore = TerrainMatch + FoodAvailability + ShelterAvailability - PredatorPressure

## 6.4 生态变量 `Long-term`

食物量、尸体量、污染值、恐慌值等持续模拟。

---

# 7. 反馈与可视化

## 7.1 Brain View `MVP`

分两层：

**玩家层（默认）**：自然语言归因 + 最多 3 个高亮 Tag。

```
选择了【绕路】
原因：记得左路有威胁（最强因素）、谨慎度偏高
```

**开发者层（F1）**：原始 Utility 数字和规则触发详情。

## 7.2 涌现高亮 `MVP`

当 AI 做出非显而易见但可解释的行为时，系统主动提示：

```
聪明行为：它没有走短路，因为它记得左路有威胁。
机会主义：它顺手捡了侧路的额外资源。
灵活撤退：它打了两口发现打不过，带伤撤退了。
```

判定条件（可硬编码）：
- AI 选择了非默认路线（因 Memory/Signal/Hazard 修正）
- AI 因 Memory 改变路线
- AI 因 Signal 改变目标
- AI 因可选资源偏离主路线

**这是爽感的核心来源。**

## 7.3 因果报告 `MVP: 摘要层` `Post-MVP: +完整层`

**摘要层**（默认显示，3-5 个关键节点）：

```
你加装了 smell_sensor
↓
00:12 检测到左路威胁气味
↓
00:14 AI 放弃短路，改走长路
↓
路径耗时增加 28 秒
↓
任务超时失败
```

**完整层**（展开查看）：SensorEvent / DecisionEvent / ActionEvent / OutcomeEvent 完整时间线。

## 7.4 Attempt Diff `MVP`

每次任务结束后和上次同地图比较：

```
与上次相比：

+ smell_sensor 在 00:12 发现威胁
+ AvoidThreat 分数提高 34
+ AI 从短路改走长路
+ 受伤次数从 3 次降到 0 次
- 总路径时间增加 28 秒
- 最终因超时失败
```

**这比单局报告更重要。** 它直接回答"我改了什么导致了什么变化"。

## 7.5 实验评分 `MVP`

每局结束给标签：

```
险胜 / 高效回收 / 过度谨慎 / 贪婪致死 / 聪明绕路 / 成功撤离
```

比纯数字报告更有情绪回报。

## 7.6 快速测试 `MVP`

30 秒 Quick Test，包含 3 个短场景：

```
1. 前方出现敌人 → 攻击/绕行/逃跑
2. 左侧出现资源 → 拾取/忽略/标记后返回
3. 短路有危险区 → 穿越/绕路/停顿
```

旋钮调完立即看到效果，不需要跑完整任务。

## 7.7 任务中反馈 `MVP: 最小`

头顶意图图标、当前状态颜色、恐惧/饥饿条、犹豫提示"？"。

## 7.8 回放系统 `Long-term`

任务录像、时间线、关键决策节点、单步查看。

---

# 8. 战斗系统

## 8.1 战斗模式 `MVP: 接触威胁` `Post-MVP: +捕食` `Long-term: +其余`

| 模式 | 阶段 |
|------|------|
| 接触威胁（碰到就受伤，可逃跑） | `MVP` |
| 捕食战斗（Bite/Charge + HP/Death） | `Post-MVP` |
| 领地 / 环境 / 群体 / 部件破坏 | `Long-term` |

## 8.2 攻击类型 `Post-MVP: Bite/Charge` `Long-term: +其余`

MVP 只做接触威胁：敌人碰到玩家 → 受伤 → 可逃跑。

Post-MVP 加 Bite（Sphere query + Damage）+ Charge（Capsule sweep + 击退）。

---

# 9. 进度系统 `Post-MVP`

## 9.1 Data 资源 `MVP: 最小`

**失败也能给 Data**。Data 奖励曲线：
- 新失败模式：大量 Data
- 重复失败模式：递减
- 防滥用：不能无脑送死刷 Data

## 9.2 三种资源 `Post-MVP`

Biomass（孵化/修复）/ Alloy（机械部件/基地）/ Data（AI 模块/蓝图）

## 9.3 科技树 / 实验站升级 `Post-MVP`

---

# 10. 游戏模式 `MVP: 2种` `Post-MVP: +`

| 模式 | 阶段 |
|------|------|
| **实验场**（安全测试，无永久损失） | `MVP` |
| **野外任务**（有目标/风险/收益） | `MVP` |
| 生态沙盒 / 斗兽场 / 远征 Roguelike | `Post-MVP` / `Long-term` |

---

# 11. 美术方案 `MVP: 占位体`

MVP 全用占位几何体，颜色/图标区分部件。但必须有：
- 发现威胁：短促警觉音
- 改路：路线变色
- 成功绕敌：高亮提示
- 带样本返回：明确完成音效

---

# 12. 关键风险

| 风险 | 解决 |
|------|------|
| 涌现 vs 噪声 | Brain View 自然语言归因 + 涌现高亮 + Attempt Diff |
| 任务中只能看戏 | 信号 cooldown + 犹豫提示 + 决策密度目标 |
| 设计粒度太粗 | 行为旋钮（5 个带张力）代替纯 Profile |
| 反馈循环太慢 | Quick Test（30 秒）+ Fast-forward（4x/8x）+ Retry Same Seed |
| 构筑有最优解通杀 | 预算约束 + 部件×环境交互 + 歧义地图 |
| AI 黑箱 | 因果报告 + Attempt Diff + Memory 事件可追溯 |
| 认知负担太高 | 预设 Profile → Quick Test → Attempt Diff 三层降低 |
| 确定性回放 hash 回归 | 尽早自动化 Replay hash CI |

---

# 13. 长线扩展 `Long-term`

繁殖/基因、玩家分享蓝图、Mod 支持、对抗式生态、完整战斗系统、程序化身体建模。
