# MAP TEMPLATES — 地图与生态区规范

> 版本 3.2 | 2026-05-28
>
> 地图是**名词容器**。每个生态区是 Run 中的一个节点。
> 房间只定义地形 + 活物的 EmitterDef/SensorDef/PersonalityBias，不定义事件。
> 事件从总线涌现，不由设计师编写。
>
> 冲突裁决：System_Core > 本文档。

---

# 1. 设计原则

## 1.1 房间是名词容器

```
房间 = 地形 + 活物配置 + 环境元素
活物配置 = EmitterDef + SensorDef + PersonalityBias（名词，不是事件）
```

设计师摆名词，总线自己跑出故事。

## 1.2 Run 结构

```
Run = 3-5 个连续生态区
每个生态区 = 3-5 个房间（随机拼接）
```

每次 Run 的房间组合不同。程序化拼接保证重玩性。

## 1.3 活物密度

每个房间至少 3-4 个独立 AI 主体 + 1-2 种环境元素。

活物的行为（发射 Stimulus → 形成 Belief → Utility 输出 → 新动作 → 新 Stimulus）自然产生事件。

---

# 2. 生态区

每个生态区是一张"考试卷"，测试不同的生物设计：

| 生态区 | 测试什么 | Run 位置 | 难度 |
|--------|---------|---------|------|
| **废弃温室** | 基础生存、路径选择 | 第 1 区 | T1 |
| **酸性洞穴** | 抗性、路径选择 | 第 2 区 | T2 |
| **机械废墟** | 潜行、电子干扰 | 第 3 区 | T2-T3 |
| **火山地带** | 热抗性、逃跑逻辑 | 第 4 区 | T3 |
| **异常区** | 未知规则 | 第 5 区 | T3+ |

## 2.1 废弃温室

```
活物配置（名词）：
- 草食小虫：EmitterDef{Visual, Prey, 0.3}，SensorDef{Visual, Range=5}
- 中型捕食者：EmitterDef{Visual, Threat, 0.8}，SensorDef{Visual+Smell, Range=8}
- 清道夫：EmitterDef{Smell, Corpse, 0.0}（只读不发），SensorDef{Smell, Range=10}

环境元素：
- 普通地面（无 Stimulus）
- 窄通道（地形约束）
- 巢穴（周期发 Smell:Food）
- 水源（周期发 Smell:Food）
```

## 2.2 酸性洞穴

```
活物配置：
- 酸蚀虫：EmitterDef{Smell, Hazard, 0.5}（自带酸），SensorDef{Smell, Range=6}
- 大型捕食者：EmitterDef{Visual+Smell, Threat, 0.9}，SensorDef{Visual+Smell, Range=10}
- 清道夫：同上

环境元素：
- 酸池：EmitterDef{Smell, Hazard, 0.7, Period=1.0}（持续发射，扩散）
- 黑暗区（Visual Range 缩小 50%）
- 矿物资源点：EmitterDef{Smell, Resource, 0.3}
```

## 2.3 机械废墟

```
活物配置：
- 机械巡逻者：EmitterDef{Visual+EM, Threat, 0.6}，SensorDef{Visual+EM, Range=8}
- 小型无人机：EmitterDef{Visual, Threat, 0.2}，SensorDef{Visual, Range=12}（侦察型）
- 异常体：特殊 EmitterDef（规则特殊）

环境元素：
- 可开门/关门（改变视觉传播路径）
- 电子干扰区（EM 通道 Range 缩小）
- 金属资源点：EmitterDef{EM, Resource, 0.4}
```

---

# 3. 房间模板

```csharp
public sealed class RoomTemplate
{
    public string TemplateId;
    public RoomCategory Category;       // Explore / Encounter / Resource / Transition
    public Vec2Int Size;                // 典型 20×20
    public List<ConnectionPort> Ports;
    public List<AgentSpawn> Spawns;     // 活物名词配置
    public List<EnvEntity> EnvEntities; // 环境元素名词配置
    public List<StaticObstacle> Obstacles;
    public List<ResourceSpot> Resources;
}

// AgentSpawn 只携带名词，不携带事件
public sealed class AgentSpawn
{
    public AgentType Type;
    public Vec2 SpawnAreaCenter;
    public float SpawnAreaRadius;
    public EmitterDef[] Emitters;       // 该活物发射什么 Stimulus
    public SensorDef[] Sensors;         // 该活物读取什么 Stimulus
    public float[] PersonalityBias;     // 性格偏置（喂进效用公式）
}
```

---

# 4. 程序拼接

每个生态区由 3-5 个房间模板拼接：

```
1. 选择入口房间
2. 程序选择 2-3 个中间房间（随机）
3. 选择出口房间
4. 连接：上一房间出口 → 下一房间入口
5. 在房间内程序化放置活物（按 AgentSpawn 配置）
```

**重玩性靠变体 + 拼接。** 地形固定，活物初始位置和环境元素随机。

---

# 5. 撤退点

每个生态区出口前有撤退选项：
- 选择"继续" → 进入下一个生态区
- 选择"撤退" → Run 结束，带回所有收集的资源

**撤退是策略核心。** 继续深入风险更高但奖励更好。
