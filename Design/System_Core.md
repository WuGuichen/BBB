# SYSTEM CORE — 刺激·信念·效用总线

> 版本 0.1 | 2026-05-28
>
> 这是项目的架构地基。所有活物与环境元素都通过这一条总线收发信号。
> 涌现(世界自己长故事)与策略(玩家用标签当杠杆)都从这里产生。
> 冲突裁决:本文档高于 Sensory_Design / World_Simulation / GDD,低于 TECHNICAL / MILESTONES。

---

## 0. 一句话

> 万物皆 **发射器 + 接收器**。动作发射刺激 → 环境传播 → 被感官过滤成信念 → 信念×需求×性格算出效用 → 产生新动作 → 再次发射刺激。闭环。

---

## 1. 两条铁律(开发红线)

**铁律 A — 无暗骰子。** 结果结算永不掷骰。随机性只允许存在于"初始位置 + 活物目标"。一只生物为何打/跑/死,必须 100% 可追溯到玩家本可知道的原因。

**铁律 B — 无实体特判。** 禁止任何 `if (对方是清道夫) 则…` 形式的交互代码。交互只能通过"发射某 tag 的刺激 / 读取某 channel"产生。带具体类型判断的交互 = 涌现杀手,code review 一律打回。

---

## 2. 正交词汇表(刻意保持小)

| 维度 | 词汇 | 约束 |
|------|------|------|
| 通道 Channel | 视觉 / 嗅觉 / 听觉 / 震动 / 热 / 电磁 | ≤6,MVP 只开 视觉+嗅觉 |
| 标签 Tag | 威胁 / 猎物 / 食物 / 同类 / 尸体 / 资源 / 危险 | ≤8 |
| 需求 Need | 饥饿 / 疲劳 / 恐惧 / 好奇 / 攻击欲 / 思乡 | 沿用 GDD §4.5 |

词汇一多就不再正交,组合失去意义。**克制是涌现的前提。**

---

## 3. 核心数据结构

```csharp
public enum StimChannel { Visual, Smell, Sound, Vibration, Heat, EM }
public enum StimTag     { Threat, Prey, Food, Kin, Corpse, Resource, Hazard }

// 任何动作/实体发射的一条刺激
public readonly struct Stimulus
{
    public StimChannel Channel;
    public StimTag      Tag;
    public float        Intensity;   // 0-1，源头强度
    public Vec2         Origin;
    public float        Radius;      // 传播半径
    public int          SourceId;
    public float        EmittedAt;   // 时间戳
}

// 实体如何发射（设计师只填这个）
public sealed class EmitterDef
{
    public StimChannel Channel;
    public StimTag      Tag;
    public float        Intensity;
    public float        Period;      // 0=单次, >0=周期发射（酸池/腐尸）
}

// 感官如何读取（决定生物会形成什么样的信念）
public sealed class SensorDef
{
    public StimChannel Channel;
    public float        Range;
    public float        AngleError;     // 方向误差，视觉≈0，嗅觉±30°
    public float        ConfidenceBase; // 视觉1.0 / 嗅觉0.6 / 听觉0.4
    public bool         GivesPosition;  // 视觉 true，嗅觉/听觉 false
    public bool         GivesIdentity;  // 能否确定"是谁/什么 tag"
}

// 生物对世界的信念（它对此行动，而非对真相行动）
public sealed class Belief
{
    public StimTag      Tag;
    public Vec2         EstimatedPos;
    public float        PosError;     // 位置不确定度
    public float        Confidence;   // 0-1
    public float        LastUpdated;  // 用于衰减
    public StimChannel  Source;
    public int?         SubjectId;    // 可能为 null（知道有威胁，不知道是谁）
}
```

## 3.5 部件运行时状态（损伤如何进入总线）

部件损伤不是特效，是**数据**。损伤改变 SensorDef/EmitterDef/Attribute，直接影响 Belief 形成和 Utility 计算。

```csharp
public enum PartImpairment
{
    None,
    Weakened,      // 功能下降
    Disabled,      // 功能失效
    Detached,      // 部件脱落
    Jammed,        // 卡住（机械体）
    Leaking,       // 漏气/漏液（自身变 Emitter）
    Misreading     // 误读（Sensor 置信度下降）
}

public sealed class PartRuntimeState
{
    public int PartId;
    public float Integrity01;           // 1.0→0.0
    public PartImpairment Impairment;
    public AttributeModifier[] OutputModifiers;  // 对属性的修正
    public EmitterDef[] DamageEmitters;          // 损伤产生的新 Stimulus（漏气/流血）
    public SensorNoiseModifier[] SensorNoise;    // 对 SensorDef 的噪声修正
}
```

**损伤→总线的映射**：

| 部件 | 损伤 | 对总线的影响 |
|------|------|------------|
| 腿 Weakened | MoveSpeed -30% | 移动发的 Visual Stimulus Intensity 降低 |
| 眼 Misreading | Visual ConfidenceBase -0.2 | 形成的 Belief 更不确定 |
| 嗅觉腔 Leaking | 自身持续发 Smell:Prey/Corpse 弱刺激 | 捕食者更容易追踪它 |
| 爪 Disabled | AttackAttribute -X | 战斗接触伤害降低 |
| 甲破裂 | DefenseAttribute -X | 下次接触伤害更高 |

**关键**：损伤产生新的 EmitterDef（漏气/流血），重新进入总线。**闭合性测试**：受伤→发出新刺激→被其他实体读到→产生新行为。

---

## 4. 传播规则(每通道一套,这是世界的物理)

| 通道 | 传播 | 被什么改变 |
|------|------|-----------|
| 视觉 | 直线,被墙/遮挡完全阻断 | 黑暗缩短 Range;遮挡=0 |
| 嗅觉 | 顺风扩散,绕过障碍 | 风向偏移 Origin;**强气味(酸雾)盖过弱气味** |
| 听觉 | 球面衰减,可绕墙(减弱) | 安静环境 Range↑ |
| 震动 | 沿地面,被空隙阻断 | — |
| 热 | 短程,直线 | 能消歧"活物 vs 无热危险" |
| 电磁 | 机械体专属 | — |

---

## 5. 信念生命周期(系统的记忆,也是"过期/错误/缺失"的来源)

1. **形成**:刺激进来 → 同 tag+近位置 → 更新已有信念;否则新建。`Confidence = SensorDef.ConfidenceBase × Stimulus.Intensity`。
2. **衰减**:`Confidence(t) = Confidence₀ × exp(-λ_channel × Δt)`。视觉 λ 小(记得久)、嗅觉 λ 大(很快不确定)。
3. **过期**:`Confidence < 阈值` → 信念删除。但"记忆衰减"渲染层(Sensory_Design §1.1)仍可显示**最后已知位置的残影**——这就是"你以为它还在原地"的张力来源。
4. **消歧 / 冲突**:
   - 多通道命中同一 SubjectId → Confidence 提升、合并。
   - 同位置出现冲突 tag(嗅觉:威胁 vs 热:小型活物)→ **不强行裁决**,二者都保留 → 直接喂给第 6 节,**这就是"犹豫"的根因**。

> **错误信念三态都从这里自然产生:过期(衰减没更新)/ 错误(嗅觉给不出 identity,把清道夫当威胁)/ 缺失(没有 sensor 覆盖该通道)。** 这三态是策略的核心战场,不要去"修复"它们。

---

## 6. 效用与犹豫(决策,且可归因)

```csharp
// 对每个候选 Intent 算分
Utility(intent) = Σ_beliefs(
      Relevance(intent, belief.Tag)   // 该信念对该意图的相关性
    × belief.Confidence               // 越不确定，影响越小
    × NeedWeight[relatedNeed]         // 内部需求放大/抑制
    × PersonalityBias[intent]         // 性格 = 固定偏置
);
// 选 argmax。每一项都可被 Decision Trace 拆开显示 → 满足铁律 A。
```

**犹豫触发**(沿用 Sensory_Design §5.2):最高分与次高分差 < 15 且 最高分 > 30 且 距上次犹豫 > 5s。犹豫不是表演,是"心智在此处境无明确答案"——它对玩家是**纯信息**。

---

## 7. 动作→刺激 映射表(保证"闭合性",链条才接得下去)

| 动作/状态 | 发射的刺激 |
|-----------|-----------|
| 移动 | 视觉:依速度强度 / 震动 |
| 冲刺/战斗 | 听觉:战斗 + 视觉 |
| 进食 | 听觉:进食 + 视觉 |
| 受伤 | 嗅觉:尸体(低强度,代表血) |
| 死亡 | 生成尸体实体 → 持续发 嗅觉:尸体 |
| 酸池/危险 | 持续发 嗅觉:危险 |
| 资源点 | 持续发 嗅觉:资源(弱) |

**每个后果都重新进总线** → 故事 C 那种"草食者死→尸体→引来清道夫→引走巡逻者"的连锁才会自己发生。

---

## 8. 两个验收测试(每加一个实体/规则就过一遍)

- **对称性测试**:它发射的刺激,能否被所有相关接收器用同一套通道规则读到,**无特判**?
- **闭合性测试**:它的动作后果,是否作为新刺激重新进入总线?

两关都过 = 会长内容;有一关靠特判 = 永远只有你写的那条。

---

## 9. 涌现 × 策略 的统一(本系统存在的理由)

- **涌现**:设计师只摆名词(EmitterDef + SensorDef + 性格权重),换初始种子就长出结构不同的故事(满足 M1"3 次显著不同")。
- **策略**:玩家唯一的笔是**生物的标签**(装哪些 SensorDef、性格权重、AI 规则)。因为一切在同一总线上,改一个标签会顺着因果图传播、重写所有下游故事;又因铁律 A 可归因,这种改写是**可推理的策略**而非赌博。

> 玩家不在"此刻"操作,而在"生成器参数"上操作。这是本项目区别于一切同类的根。

---

## 10. 设计师工作边界

✅ 调词汇表 / 写 EmitterDef + SensorDef / 摆名词 / 调性格权重
❌ 写事件 / 写时间线 / 写"谁对谁做什么" / 写实体特判
