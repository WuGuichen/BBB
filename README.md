# WGame_B

基于 MxFramework（WGameFramework）开发的 AI 生物行为实验游戏。

> **调教一个半自主 AI 生物，理解它为什么这么做，从失败中获得可执行的知识。**

## 当前状态

**阶段**：概念期末尾 / 预制作期入口

**当前目标**：Behavior Lab Slice v0 — 验证"AI 行为调教是否好玩"

本阶段不验证生态模拟、不验证战斗系统、不验证程序化建模。只验证：
- 玩家能不能通过旋钮调教 AI 行为？
- AI 行为差异是否可感知、可解释？
- 失败是否能给玩家可执行的知识？
- "涌现时刻"是否能产生爽感？

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

1. 已实现代码和测试
2. MILESTONES 当前阶段
3. TECHNICAL 技术约束
4. GDD 玩法意图
5. CONCEPT 高层愿景

## 文档

设计文档位于 `Design/`，阅读顺序见 [Design/README.md](Design/README.md)。

## 框架基线

- 框架仓库：`http://192.168.1.210:3002/vincent/WGameFramework`
- 框架能力清单：`WGameFramework/Docs/CAPABILITIES.md`
- 游戏开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
