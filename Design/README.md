# WGame_B 设计文档

> 版本 3.2 | 2026-05-28

## 阅读顺序

| 顺序 | 文档 | 用途 |
|------|------|------|
| 1 | [System_Core.md](System_Core.md) | 刺激·信念·效用总线（地基） |
| 2 | [Sensory_Design.md](Sensory_Design.md) | 感知系统（总线的渲染层） |
| 3 | [World_Simulation.md](World_Simulation.md) | 多智能体仿真（总线的名词库） |
| 4 | [Map_Templates.md](Map_Templates.md) | 地图与生态区 |
| 5 | [GDD.md](GDD.md) | 游戏设计文档 |
| 6 | [TECHNICAL.md](TECHNICAL.md) | 技术方案 |
| 7 | [MILESTONES.md](MILESTONES.md) | 制作计划 |
| 8 | [CONCEPT.md](CONCEPT.md) | 概念定位 |

## 裁决链

```
代码 > MILESTONES > TECHNICAL > System_Core > Sensory_Design ≈ Map_Templates > GDD > CONCEPT
```

## 铁律

- **铁律 A — 无暗骰子。** 结果 100% 可追溯。
- **铁律 B — 无实体特判。** 一切交互通过总线产生。

## 原则

- 每份文档只写一次
- 设计师只摆名词，总线自己跑出故事

## 关联

- 框架：`WGameFramework/Docs/CAPABILITIES.md`
- 开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
