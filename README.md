# WGame_B

一款 Creature Builder Roguelite。设计生物的身体和AI，投放到危险生态区，观察它们自主行动、失败、成功，回收资源继续改造。

> **设计身体，编排行为，投放生态，观察结果，回收改造。**

## 当前状态

**阶段**：概念期 → 预制作期

**版本**：2.0（2026-05-28）

**当前目标**：M1 判生死验证

玩家能组装生物、配置简单AI、投放观察。同一房间3次播放事件不同。

## 更接近什么游戏

**Cogmind × Screeps × Spore × RimWorld**

## 核心设计

- **身体部件**：8类部件（Core/Movement/Sensor/Weapon/Defense/Utility/Brain/Social）
- **AI蓝图**：卡片式规则，玩家配置行为
- **感知**：视觉+嗅觉（MVP），听觉（Post-MVP）
- **涌现**：多智能体+环境独立+需求系统+跨Run持久化
- **信号**：有限信号，不直接控制
- **任务**：回收/观察/狩猎/护送/清理
- **实验站**：部件仓库/AI台/科技树
- **战后报告**：死亡原因/建议修改

## 命名关系

| 名称 | 含义 |
|------|------|
| **WGameFramework** | 游戏框架（Gitea: `vincent/WGameFramework`） |
| **MxFramework** | 框架内部名称 |
| **VVV** | WGameFramework 的 GitHub 镜像名称（即框架代码本身） |
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
