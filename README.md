# WGame_B

> **设计一个系统，让它自己跑。世界测试你的设计，失败给你数据，改造后再来。**

## 两条铁律

- **铁律 A — 无暗骰子。** 结果 100% 可追溯。没有隐藏 RNG。
- **铁律 B — 无实体特判。** 一切交互通过总线的刺激/信念/标签产生。

## 裁决链

```
代码 > MILESTONES > TECHNICAL > System_Core > Sensory_Design ≈ Map_Templates > GDD > CONCEPT
```

## 文档（阅读顺序）

### 地基

1. [System_Core.md](Design/System_Core.md) — 刺激·信念·效用总线

### 核心

2. [Sensory_Design.md](Design/Sensory_Design.md) — 感知系统（总线的渲染层）
3. [World_Simulation.md](Design/World_Simulation.md) — 多智能体仿真（总线的名词库）
4. [Map_Templates.md](Design/Map_Templates.md) — 地图与生态区

### 框架

5. [GDD.md](Design/GDD.md) — 游戏设计文档
6. [TECHNICAL.md](Design/TECHNICAL.md) — 技术方案
7. [MILESTONES.md](Design/MILESTONES.md) — 制作计划
8. [CONCEPT.md](Design/CONCEPT.md) — 概念定位

## 命名关系

| 名称 | 含义 |
|------|------|
| **WGameFramework** | 游戏框架（Gitea: `vincent/WGameFramework`） |
| **MxFramework** | 框架内部名称 |
| **VVV** | WGameFramework 的 GitHub 镜像名称 |
| **WGame_B** | 游戏设计项目（Gitea: `vincent/WGame_B`） |
| **WGame** | 游戏层代码目录（基于 MxFramework 实现） |
