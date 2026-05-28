# WGame_B

一款 Creature Builder Roguelite。设计生物的身体和AI，带它们深入危险生态区，活着带回尽可能多的东西。

> **设计身体，编排行为，深入生态，死或撤退，解锁重来。**

## 当前状态

**阶段**：概念期 → 预制作期

**版本**：3.0（2026-05-28）

**当前目标**：M4 判生死验证

3个连续生态区 + 撤退决策 + Run内部件/样本分离。

## 核心问题

**这次能走多远？**

不是"我收集了什么"，是"这次Run我能深入到第几区"。

## 更接近什么游戏

**FTL × Spelunky × Cogmind × Screeps**

## 核心设计

- **身体部件**：8类（Core/Movement/Sensor/Weapon/Defense/Utility/Brain/Social）
- **AI蓝图**：卡片式规则，玩家配置行为
- **Run结构**：3-5个连续生态区，难度递增
- **撤退**：随时可撤退，带回当前收集。全灭丢失Run内资源
- **Meta进度**：样本/数据死了保留，用于解锁新选项
- **涌现**：多智能体+环境独立+需求系统

## 命名关系

| 名称 | 含义 |
|------|------|
| **WGameFramework** | 游戏框架（Gitea: `vincent/WGameFramework`） |
| **MxFramework** | 框架内部名称 |
| **VVV** | WGameFramework 的 GitHub 镜像名称 |
| **WGame_B** | 游戏设计项目（Gitea: `vincent/WGame_B`） |
| **WGame** | 游戏层代码目录（基于 MxFramework 实现） |

## 文档

- [CONCEPT.md](Design/CONCEPT.md) — 概念定位
- [GDD.md](Design/GDD.md) — 游戏设计文档
- [Sensory_Design.md](Design/Sensory_Design.md) — 感知系统设计规范
- [World_Simulation.md](Design/World_Simulation.md) — 多智能体仿真与生态规范
- [Map_Templates.md](Design/Map_Templates.md) — 地图与生态区规范
- [TECHNICAL.md](Design/TECHNICAL.md) — 技术方案
- [MILESTONES.md](Design/MILESTONES.md) — 制作计划
