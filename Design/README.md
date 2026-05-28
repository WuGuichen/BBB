# WGame_B 设计文档

## 阅读顺序

| 顺序 | 文档 | 版本 | 用途 |
|------|------|------|------|
| 1 | [CONCEPT.md](CONCEPT.md) | v0.8 | 一页纸概念 |
| 2 | [Sensory_Design.md](Sensory_Design.md) | v0.8 | 感知与未知设计规范（核心设计文档） |
| 3 | [GDD.md](GDD.md) | v0.8 | 玩法设计 |
| 4 | [TECHNICAL.md](TECHNICAL.md) | v0.8 | 技术方案 |
| 5 | [MILESTONES.md](MILESTONES.md) | v0.8 | 制作计划 |

## v0.8 主要变更

核心方向从"自走棋"改为"陪伴探险"：

- **核心动词变了**：从"组装/摆位置"改为"通过他者的感知理解未知"
- **新增 Sensory_Design.md**：三层信息模型、感知模态视觉语言、信息密度要求
- **感知系统前置**：视觉+嗅觉是 MVP 核心，不是后期功能
- **性格深度**：矛盾叠加 + 短期记忆 + 信任度
- **战斗降级**：从"主要事件"降为"探险中的一个环节"（3-5秒短接触）
- **干预有人际感**：警告/建议/安抚/鼓舞，每个对不同性格响应不同
- **里程碑重排**：M1 感知迷雾、M2 性格犹豫是判生死节点
- **竞品校准**：Pikmin / Lemmings / Darkest Dungeon / Lifeline

## 原则

- 每份文档只写一次
- 冲突裁决：代码 > MILESTONES > TECHNICAL > GDD > CONCEPT > Sensory_Design

## 关联

- 框架：`WGameFramework/Docs/CAPABILITIES.md`
- 开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
