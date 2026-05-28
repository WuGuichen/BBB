# WGame_B 设计文档

## 阅读顺序

| 顺序 | 文档 | 版本 | 用途 |
|------|------|------|------|
| 1 | [CONCEPT.md](CONCEPT.md) | v0.9 | 一页纸概念 |
| 2 | [Sensory_Design.md](Sensory_Design.md) | v0.9 | 感知与未知设计规范（核心设计文档） |
| 3 | [GDD.md](GDD.md) | v0.9 | 玩法设计 |
| 4 | [Map_Templates.md](Map_Templates.md) | v0.9 | 房间模板规范（10 个 MVP 模板 + 节拍清单 + 拼接规则） |
| 5 | [TECHNICAL.md](TECHNICAL.md) | v0.9 | 技术方案 |
| 6 | [MILESTONES.md](MILESTONES.md) | v0.9 | 制作计划 |

## v0.9 主要变更

基于"陪伴探险"方向的第一次专业评审收口：

- **裁决顺序修正**：Sensory_Design 提到 GDD 之前。核心动词的描述优先于玩法系统的描述。
- **MVP 锁定单生物**：陪伴一只生物的情感产品优先级高于多生物的注意力分配玩法。多生物明确移到 Post-MVP，不再作为 MVP 验证项。
- **Trust v0 表现层前置到 M3**：信任关系是"陪伴"幻想的支柱。系统层留在 Post-MVP，但表现层（听话/没听/迟疑三种反馈短语）必须在早期 playtest 中可见。
- **新增 M1 验收剧本**：60 秒首房脚本写入 Sensory_Design §7，作为 M1 验收的剧本而非"附加文档"。
- **AI 反馈分四层**：行为 / 情绪 / 理由 / 结果。写入 Sensory_Design §4。
- **干预早期不测稀缺性**：M4 固定 2-3 次喊话，先测"沟通感"；零件经济和稀缺性留到 M7。
- **性格表现三件套**：每个 AI 模块定义 微行为 + UI 语言 + 音效倾向。
- **新增 Decision Trace Overlay**：开发期调试系统，写入 TECHNICAL §9。
- **新增 Sensory Sandbox**：M1 配套的调试场景，可重置/无限信号/切换 build。
- **新增"生物作为角色"**：命名、伤痕、归来、纪念、Run 总结——把"AI 单位"变成"我的生物"。
- **明确 2.5D 等距视角**：写入 TECHNICAL §11，避免 M1 时相机方向漂移。
- **新增 Map_Templates.md**：10 个 MVP 房间模板（含 M1 验收剧本专用的 R01-Lab）、节拍元素清单（15 种）、程序拼接规则、节拍密度自检规范。

## 原则

- 每份文档只写一次
- **冲突裁决**：代码 > MILESTONES > TECHNICAL > Sensory_Design ≈ Map_Templates > GDD > CONCEPT
  - Sensory_Design 和 Map_Templates 同级：前者管"看到什么"，后者管"在哪看到"

## 关联

- 框架：`WGameFramework/Docs/CAPABILITIES.md`
- 开发规范：`WGameFramework/Docs/AGENT_GAME_CREATION_GUIDE.md`
