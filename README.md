# WGame_B

一款 Creature Builder Roguelite。设计生物的身体和AI，带它们深入危险生态区，活着带回尽可能多的东西。

> **设计身体，编排行为，深入生态，死或撤退，解锁重来。**

## 当前状态

**阶段**：概念期 → 预制作期

**版本**：3.1（2026-05-28）

**当前目标**：M4 判生死验证

3个连续生态区 + 撤退决策 + Run内部件/样本分离。

## 核心问题

**这次能走多远？**

## 更接近什么游戏

**FTL × Spelunky × Cogmind × Screeps**

## 铁律

- **铁律 A — 无暗骰子。** 结果结算永不掷骰。随机性只允许存在于"初始位置 + 活物目标"。
- **铁律 B — 无实体特判。** 禁止 `if (对方是清道夫)` 形式的交互代码。交互只能通过总线产生。

## 裁决链

```
代码 > MILESTONES > TECHNICAL > System_Core > Sensory_Design ≈ Map_Templates > GDD > CONCEPT
```

## 命名关系

| 名称 | 含义 |
|------|------|
| **WGameFramework** | 游戏框架（Gitea: `vincent/WGameFramework`） |
| **MxFramework** | 框架内部名称 |
| **VVV** | WGameFramework 的 GitHub 镜像名称 |
| **WGame_B** | 游戏设计项目（Gitea: `vincent/WGame_B`） |
| **WGame** | 游戏层代码目录（基于 MxFramework 实现） |

## 文档（阅读顺序）

### 地基

1. [System_Core.md](Design/System_Core.md) — 刺激·信念·效用总线（架构地基）

### 核心

2. [Sensory_Design.md](Design/Sensory_Design.md) — 感知系统（总线的渲染层）
3. [World_Simulation.md](Design/World_Simulation.md) — 多智能体仿真（总线的名词库）
4. [Map_Templates.md](Design/Map_Templates.md) — 地图与生态区

### 框架

5. [GDD.md](Design/GDD.md) — 游戏设计文档
6. [TECHNICAL.md](Design/TECHNICAL.md) — 技术方案
7. [MILESTONES.md](Design/MILESTONES.md) — 制作计划
8. [CONCEPT.md](Design/CONCEPT.md) — 概念定位
