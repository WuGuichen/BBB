# WGame_B 设计文档

## 阅读顺序

| 顺序 | 文档 | 版本 | 用途 | 读者 |
|------|------|------|------|------|
| 1 | [CONCEPT.md](CONCEPT.md) | v0.3 | 一页纸概念定位 | 任何人 |
| 2 | [GDD.md](GDD.md) | v0.3 | 玩法设计正文（每系统标 MVP/Post-MVP/Long-term） | 策划、程序、美术 |
| 3 | [TECHNICAL.md](TECHNICAL.md) | v0.3 | 技术方案、框架映射、缺口分析、游戏层临时实现策略 | 程序 |
| 4 | [MILESTONES.md](MILESTONES.md) | v0.3 | 制作计划（M0-M9，量化验收条件） | 制作人、程序 |

## 原则

- **每份文档只写一次**。交叉引用代替复制粘贴。
- **设计决策和理由**归 GDD；**框架映射和配置格式**归 TECHNICAL；**排期和验收**归 MILESTONES。
- **冲突裁决**：代码 > MILESTONES 当前阶段 > TECHNICAL 约束 > GDD 意图 > CONCEPT 愿景
- 旧草稿已归档至 `Archive/`。

## v0.4 主要变更（2026-05-27）

基于第二轮审阅（设计层面）修订：

- **CONCEPT**：补竞品（The Bibites / The Sapling / Creatures）；补核心风险"涌现 vs 噪声"；补目标玩家画像
- **GDD**：Brain View 改为双层（玩家层自然语言归因 + 开发者层原始数字）；Data 奖励加递减曲线防滥用；风险表加"涌现 vs 噪声"和"构筑通杀"
- **MILESTONES**：M3/M6/M7 加主观 Playtest 节点；M5 加反协同验证；M6/M7 完成后立即跑批量仿真（不等到 M9）；M8 信号效果数字改为"批量仿真后标定"；风险表加 hash 回归和构筑通杀
- **TECHNICAL**：新增 §7 确定性回放代价，强调 Replay hash CI 自动化

## v0.3 主要变更（2026-05-27）

基于制作人审阅意见修订：

- **README**：补当前状态、命名关系、冲突裁决规则
- **CONCEPT**：补"首个可玩切片"小节，隔离愿景与 MVP
- **GDD**：每个系统标阶段标签（MVP/Post-MVP/Long-term）；MVP 范围收窄（6 类部件、3 个 Profile、2 个信号、2 种攻击、1 种任务）
- **TECHNICAL**：补"游戏层先硬编码验证"策略；AI Profile 用游戏层 Adapter 不等框架扩展；MVP 导航用手工 Waypoint+BFS
- **MILESTONES**：重排为 M0-M9；M2 只用 TaskFirst；Brain View/Report 提前到 M3；验收条件量化

## 关联文档

- 框架能力清单：`WGameFramework/Docs/CAPABILITIES.md`
- 框架使用手册：`WGameFramework/Docs/USAGE.md`
- 游戏开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
