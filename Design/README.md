# WGame_B 设计文档

## 阅读顺序

| 顺序 | 文档 | 版本 | 用途 |
|------|------|------|------|
| 1 | [CONCEPT.md](CONCEPT.md) | v0.6 | 一页纸概念 |
| 2 | [GDD.md](GDD.md) | v0.6 | 玩法设计 |
| 3 | [TECHNICAL.md](TECHNICAL.md) | v0.6 | 技术方案 |
| 4 | [MILESTONES.md](MILESTONES.md) | v0.6 | 制作计划 |

## 原则

- 每份文档只写一次，交叉引用代替复制
- 冲突裁决：代码 > MILESTONES > TECHNICAL > GDD > CONCEPT

## v0.6 主要变更

核心方向从"AI 行为实验"改为"Roguelike 即时策略"：

- 零件不是自由选择，是探索中获得（稀缺性驱动策略）
- 信号是道具，不是 cooldown 能力
- 压力倒逼策略（时间、敌人、背包容量）
- AI 自动执行是为了让策略成为唯一变量
- 类比：Slay the Spire 的实时生物版

## 关联文档

- 框架能力清单：`WGameFramework/Docs/CAPABILITIES.md`
- 游戏开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
