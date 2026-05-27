# WGame_B

基于 MxFramework（WGameFramework）开发的 AI 生物构筑沙盒游戏。

> **设计身体，编排行为，投放生态，观察结果，回收改造。**

## 当前状态

**阶段**：概念期末尾 / 预制作期入口

**当前目标**：Creature Mission Slice v0 — 验证核心玩法闭环

本阶段不是完整生态沙盒，而是验证：
- 一只可配置 AI 的生物
- 在一个确定性小地图中
- 完成回收任务或失败
- 并生成可解释战报

## 核心差异化

> 不是我控制怪物战斗，而是我设计怪物的身体和脑子，让它在生态地图中证明自己。

## 命名关系

| 名称 | 含义 |
|------|------|
| **WGameFramework** | 游戏框架仓库（Gitea: `vincent/WGameFramework`） |
| **MxFramework** | 框架内部名称（代码命名空间、文档标题） |
| **WGame_B** | 游戏设计项目仓库（Gitea: `vincent/WGame_B`） |
| **VVV** | 游戏层代码目录名（`Assets/Scripts/VVV/`） |
| **BBB** | Gitea 仓库曾用名（已更名为 WGame_B） |

## 文档冲突裁决规则

当文档冲突时，按以下优先级裁决：

1. 已实现代码和测试
2. MILESTONES 当前阶段
3. TECHNICAL 技术约束
4. GDD 玩法意图
5. CONCEPT 高层愿景

CONCEPT 和 GDD 可以有远期想象，但当前做不做由 MILESTONES 和实际代码决定。

## 文档

设计文档位于 `Design/`，阅读顺序见 [Design/README.md](Design/README.md)。

## 框架基线

- 框架仓库：`http://192.168.1.210:3002/vincent/WGameFramework`
- 框架能力清单：`WGameFramework/Docs/CAPABILITIES.md`
- 游戏开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
