# WGame_B 设计文档

## 阅读顺序

| 顺序 | 文档 | 版本 | 用途 |
|------|------|------|------|
| 1 | [CONCEPT.md](CONCEPT.md) | v0.7 | 一页纸概念 |
| 2 | [GDD.md](GDD.md) | v0.7 | 玩法设计 |
| 3 | [TECHNICAL.md](TECHNICAL.md) | v0.7 | 技术方案 |
| 4 | [MILESTONES.md](MILESTONES.md) | v0.7 | 制作计划 |

## v0.7 主要变更

基于深度审阅修订 — 解决决策密度和结构性问题：

- **小队制**：从单生物改为 2-3 只小队，解决协同空间和决策密度
- **Build Identity**：生物在 Run 内持续存在，核心零件死亡不丢
- **操作维度**：信号 + 侦察 + 撤退，从 1 维变 3 维
- **分支星图**：精英/事件/休息节点，路径选择本身是策略
- **AI 模块矛盾叠加**：勇气+谨慎 = 高血量冲低血量逃，产生真正犹豫
- **战斗可观察性**：前摇+音效+伤害数字
- **竞品校准**：从 StS 改为 Super Auto Pets / Mechabellum / Backpack Hero
- **Playtest 前移**：M2 就开始给人看，不等到 M6

## 原则

- 每份文档只写一次
- 冲突裁决：代码 > MILESTONES > TECHNICAL > GDD > CONCEPT

## 关联

- 框架：`WGameFramework/Docs/CAPABILITIES.md`
- 开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
