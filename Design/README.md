# WGame_B 设计文档

## 阅读顺序

| 顺序 | 文档 | 版本 | 用途 | 读者 |
|------|------|------|------|------|
| 1 | [CONCEPT.md](CONCEPT.md) | v0.5 | 一页纸概念定位 | 任何人 |
| 2 | [GDD.md](GDD.md) | v0.5 | 玩法设计正文 | 策划、程序、美术 |
| 3 | [TECHNICAL.md](TECHNICAL.md) | v0.5 | 技术方案与框架映射 | 程序 |
| 4 | [MILESTONES.md](MILESTONES.md) | v0.5 | 制作计划 | 制作人、程序 |

## 原则

- **每份文档只写一次**。交叉引用代替复制粘贴。
- **设计决策和理由**归 GDD；**框架映射和配置格式**归 TECHNICAL；**排期和验收**归 MILESTONES。
- **冲突裁决**：代码 > MILESTONES 当前阶段 > TECHNICAL 约束 > GDD 意图 > CONCEPT 愿景
- 旧草稿已归档至 `Archive/`。

## v0.5 主要变更（2026-05-27）

基于深度玩法分析修订 — 核心循环从"观察型"改为"实验型"：

- **CONCEPT**：核心循环改为行为实验循环；MVP 改名 Behavior Lab Slice
- **GDD**：
  - 新增行为旋钮系统（5 个旋钮 + 张力关系）
  - 新增 AI Memory（3 个记忆）
  - 信号系统提前到 MVP（Avoid/Recall + cooldown）
  - Brain View 改为双层（自然语言 + 原始数字）
  - 新增涌现高亮、Attempt Diff、Quick Test、实验评分
  - Data 奖励加递减曲线
- **MILESTONES**：完全重排为 M0-M9
  - 行为深度前移（M2 旋钮、M3 Quick Test、M4 信号）
  - 部件和战斗后移（M7/M8）
  - 每个阶段加 Playtest 节点
  - 验收条件量化（决策密度、策略多样性、可解释性）
- **TECHNICAL**：
  - 新增旋钮→AI Planner 映射
  - 新增 Memory/Signal 技术方案
  - 新增 §7 确定性回放代价

## 关联文档

- 框架能力清单：`WGameFramework/Docs/CAPABILITIES.md`
- 框架使用手册：`WGameFramework/Docs/USAGE.md`
- 游戏开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
