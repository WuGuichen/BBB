# 《AI Creature Sandbox》战斗与地图设计文档 v0.1

---

# 1. 竞品分析与差异化定位

没有完全一样的成熟样板。市面上更常见的是把这几个方向拆开做：

```text
程序化运动生物：有
生态模拟：有
生物构筑：有
玩家配置 AI：少
全部合在一个游戏里：非常少
```

最接近的方向不是一个游戏，而是一组参考：

| 游戏 | 相似点 | 不足 |
|------|--------|------|
| **Ecosystem** | 虚拟生物由 DNA 生成身体、神经系统、战斗能力；生物通过关节力矩在流体中运动；有进化、捕食、地形雕刻、物种分享 | 水下生态为主，不是陆地动作/探索游戏 |
| **Rain World** | 程序化/代码驱动动画 + 捕食者/猎物生态 + 地图区域强设计 | 2D，玩家不是创造生物/AI |
| **Waking Mars** | 地图就是生态谜题，玩家通过理解物种行为来构建生态 | 不是战斗构筑游戏 |
| **Niche** | 基因、生物适应、生态位、不同 biome 中的捕食者/猎物/植物 | 回合制，不是程序化运动 |
| **Spore** | 身体编辑、能力由部件决定、从细胞到太空的生物进化幻想 | 生态和 AI 深度不够，战斗偏轻 |
| **Impossible Creatures** | 组合动物部件生成战斗单位，策略来自生物组合 | RTS，不是生态模拟/程序化运动 |

**Ecosystem** 是"程序化运动生物"最接近的参考：虚拟生命由 synthetic DNA 生成，DNA 编码骨骼结构、神经处理器和战斗能力；生物通过关节施加力矩推动水体来游动；包含战斗层（电击、毒刺、捕食关系和食物链角色）；允许玩家塑造海底地形。

**Rain World** 更接近"生态体验"：世界有超过 1600 个房间、12 个区域，玩家需要在捕食者、资源、降雨周期之间生存；slugcat 的自然流动感来自程序化生成动画，生物动画由代码和传统动画结合，使它们能柔软、可弯曲，并响应环境。

**关键事实**：不一定需要高精度物理才能做出"活的生态"，关键是地图拓扑、AI、感知、捕食关系和运动表现要一起设计。

**Waking Mars** 证明了"地图不是装饰，而是生态规则容器"这个方向是可玩的。

**Niche** 有价值的不是回合制，而是环境压力设计：物种不是凭空强弱，而是被 biome 筛选。

**Spore** 和 **Impossible Creatures** 的启发：身体部件必须映射成清晰的战斗角色和策略，不然"创造生物"只是捏模型。

---

# 2. 战斗模式定位

当前技术基础：

```text
AABB
OBB
Sphere
确定性物理
基础碰撞体
```

不适合一开始做：

```text
高精度近战判定
复杂 ragdoll
任意三角网格地形碰撞
真实爪击/咬合碰撞
高自由度攀爬
复杂软体接触
```

更适合做：

```text
意图驱动战斗
区域判定战斗
生态位战斗
地形调度战斗
捕食/逃跑/包围/伏击
```

战斗应该更像：

```text
设计生物 → 投放地图 → 生物基于 AI 和地形自主战斗 → 玩家观察和干预
```

> **战斗本质是生态策略的结果，而不是玩家手搓连招。**

---

# 3. 五种战斗模式

## 3.1 捕食战斗

最基础的战斗模式。

```text
捕食者发现猎物
↓
追踪
↓
接近
↓
冲刺 / 伏击 / 包围
↓
咬住 / 撞击 / 毒击
↓
猎物逃跑 / 反击 / 躲藏
```

胜负不靠精确 hitbox，而靠：

```text
感知能力
速度
体力
地形
恐惧
群体数量
攻击距离
逃跑路径
```

例如：

```text
六足猎兽：开阔地追击强
短腿甲虫：洞穴防守强
飞行单位：跨越地形强，但怕狭窄空间
蛇形单位：窄洞强，开阔地弱
```

命中可以用 Sphere / OBB 范围判断。

---

## 3.2 领地战斗

生物不是见面就打，而是围绕地盘和资源产生冲突。

```text
巢穴
水源
尸体
矿物
繁殖点
高地
热源
暗洞
```

战斗流程：

```text
入侵领地
↓
警告
↓
驱赶
↓
短暂冲突
↓
一方撤退
```

比"打到死"更像生态。

好处：

```text
减少无意义全灭
增加观察性
让 AI 性格有表现空间
```

例如：

```text
胆小物种：警告后退
暴躁物种：直接攻击
群居物种：呼叫同伴
伏击物种：不追击，只守住洞口
```

---

## 3.3 群体战斗

玩家可配置 AI 很适合群体战。

群体战不需要精确格斗，而需要：

```text
角色分工
站位
包围
诱饵
撤退
保护
```

例如小队：

```text
侦察者：发现目标
诱饵者：吸引注意
猎手：侧翼突袭
搬运者：不战斗，拿资源
护卫者：保护搬运者
```

战斗时：

```text
侦察者看见大型敌人
↓
发出信息素
↓
猎手从侧面绕行
↓
诱饵制造噪音
↓
搬运者趁机回收资源
```

比传统攻击动作更适合"AI 蓝图玩法"。

---

## 3.4 环境战斗

地图本身应该是武器。

```text
酸池
毒气
塌方
火焰
水流
黑暗
窄道
高台
陷阱植物
噪音区域
```

玩家不需要直接控制生物攻击，而是利用环境：

```text
放诱饵把敌人引进酸池
用噪音让捕食者离开巢穴
用火光驱散夜行生物
把群体生物引进狭窄通道，使其数量优势失效
```

环境大多可以是：

```text
AABB trigger
OBB hazard zone
Sphere influence zone
```

---

## 3.5 部件破坏战斗

做"局部伤害"，但不做真实 mesh 碰撞。

身体分成逻辑部件：

```text
Core
Head
Sensor
Legs
Weapon
Armor
Tail
Wing
Cargo
```

每个部件有自己的简单碰撞代理：

```text
Head: Sphere
Body: OBB
Leg group: AABB / OBB
Tail: chain of spheres
Wing: OBB
```

命中后不是"打中三角面"，而是：

```text
攻击范围与某个部件碰撞代理重叠
↓
根据攻击方向、护甲、部件类型结算
```

例如：

```text
咬击命中腿部 → 移速下降
酸液命中甲壳 → 护甲耐久下降
电击命中传感器 → 感知短暂失效
尾巴被打断 → 平衡下降，转向变差
翅膀受损 → 无法跨越峡谷
```

战斗有深度，而且不要求复杂物理。

---

# 4. 推荐攻击类型

## 4.1 咬击 Bite

判定：

```text
MouthSphere overlaps TargetBodySphere/OBB
且 angle < 60°
且 distance < biteRange
```

效果：

```text
高单体伤害
可能造成出血
如果目标小于自己，可以拖拽
```

适合：捕食者、伏击生物、蛇形生物、犬型生物

---

## 4.2 冲撞 Charge

判定：

```text
AttackerOBB swept forward
与 TargetOBB/Sphere 重叠
```

效果：

```text
击退
打断
撞晕
撞入危险地形
```

适合：重甲生物、大型食草者、机器人

---

## 4.3 爪击 Slash

判定：

```text
短时间激活一个前方 OBB / 扇形多 Sphere
```

效果：

```text
中距离近战
可打断轻型单位
```

适合：双足、四足、敏捷猎手

---

## 4.4 尾击 Tail Swipe

判定：

```text
尾巴轨迹用多个 Sphere 或 OBB 近似
```

效果：

```text
范围击退
防包围
弱伤害
```

适合：大型生物、三足不稳定生物、尾巴平衡型生物

---

## 4.5 毒刺 / 骨刺 Projectile

判定：

```text
Sphere projectile
```

效果：

```text
远程伤害
中毒
减速
标记目标
```

适合：脆弱远程单位、伏击单位、防御型巢穴生物

---

## 4.6 抓取 Grapple

不做真实绳索，用逻辑连接。

判定：

```text
Target within Cone/Sphere
且目标体重 < 抓取上限
```

效果：

```text
目标进入 Grabbed 状态
移动速度下降
被拖向攻击者
持续消耗攻击者体力
```

足够好玩，不需要复杂物理绳。

---

# 5. 地图设计理念

地图要"讲究"，但不是讲究美术细节，而是讲究**生态拓扑**。

地图应该由这些东西组成：

```text
可通行空间
危险空间
资源空间
巢穴空间
视野空间
声音传播空间
气味传播空间
物种分布空间
```

地图不是 mesh，而是一组语义区域。

---

# 6. 碰撞友好型地图设计

以 AABB、OBB、Sphere 为主，先避免：

```text
连续起伏山地
复杂三角网格洞穴
自由攀爬岩壁
大量不规则坡面
真实水体流体
复杂植被碰撞
```

更适合：

```text
房间制地图
模块化地形
盒状/斜盒状平台
有限坡度
明确通道
明确门户
层级高度
简单 hazard volume
```

视觉 mesh 可以复杂一点，碰撞一定要干净。

碰撞代理：

```text
AABB 地块
OBB 斜坡
Sphere 巢穴/感知/资源区域
Box tunnel
Box platform
Box wall
```

---

# 7. 推荐地图结构：Room + Portal + Habitat Zone

```text
Region
├── Room
│   ├── Collision Volumes
│   ├── Habitat Zones
│   ├── Resource Nodes
│   ├── Den Points
│   ├── Hazard Volumes
│   ├── Sight Blocks
│   └── Portals
└── Room Graph
```

## Room

一个房间是一个生态小容器。

例如：

```text
酸池洞穴
开阔废墟
暗巢通道
高台平台
管道迷宫
菌毯温室
```

## Portal

房间之间的连接。

```text
宽门
窄洞
高台跳跃口
水道
通风管
塌陷缝隙
```

Portal 决定哪些生物能通过。

例如：

```text
大型生物不能进窄洞
不会跳跃的生物不能过断层
怕水生物不能走水道
飞行生物可以跨过地面障碍
```

这是地图和物种能力的绑定。

---

# 8. 地形特征与物种分布

每个地形区域有 Habitat Tags：

```text
Light: Bright / Dim / Dark
Moisture: Dry / Damp / Wet
Temperature: Cold / Normal / Hot
Terrain: Open / Tunnel / Vertical / Water / Acid / Dense
Cover: None / Sparse / Dense
Noise: Quiet / Loud
Resource: Meat / Plant / Metal / Crystal / Corpse
Danger: Low / Medium / High
```

每个物种也有偏好：

```text
SpeciesProfile
{
    preferredLight = Dark
    preferredTerrain = Tunnel
    food = Corpse
    avoid = Fire
    denType = NarrowHole
    activity = Night
    groupSize = 3~8
}
```

分布不是随机刷怪，而是算适应度：

```text
SpawnScore =
    TerrainMatch
  + FoodAvailability
  + ShelterAvailability
  + TemperatureMatch
  + MoistureMatch
  - PredatorPressure
  - Competition
  - HumanActivity
```

物种分布会自然很多。

---

# 9. 推荐生态地图类型

## 9.1 开阔盆地

结构：大平地、少量岩石、多个资源点、少遮挡

适合：高速追击、群体围猎、大型生物、远程视野

物种：草食/采集单位、高速捕食者、群体猎手、飞行侦察者

战斗特点：逃跑路线多、伏击少、速度价值高

---

## 9.2 窄洞网络

结构：狭窄通道、小房间、多个短连接、低视野

适合：小型生物、伏击者、蛇形生物、嗅觉单位

物种：清道夫、小型毒虫、巢穴生物、钻洞单位

战斗特点：大体型受限、群体数量被通道限制、感知能力重要

非常适合基础碰撞，因为全是 OBB/AABB 通道。

---

## 9.3 垂直脚手架

结构：高低平台、斜坡、跳跃点、桥、管道

适合：跳跃生物、攀爬生物、飞行生物、长腿生物

物种：高处伏击者、飞行捕食者、攀爬采集者

战斗特点：高度差决定接触机会、不会跳跃的单位被隔离、远程攻击有价值

3D 物理简单时，垂直结构尽量用：

```text
AABB 平台
OBB 斜坡
固定高度层
明确 Portal
```

---

## 9.4 危险湿地区

结构：酸池、浅水、浮岛、安全石块、毒气区

适合：抗性构筑、路线规划、环境战斗

物种：酸抗生物、两栖生物、诱捕植物、慢速重甲

战斗特点：敌人不一定被你打死，而是被引进危险地形；抗性部件变得重要

全部用 volume 实现。

---

## 9.5 巢穴区

结构：中心巢穴、多条入口、资源存储点、守卫巡逻区、逃生小洞

适合：潜入、偷资源、引怪、群体战

物种：群居虫群、守卫型生物、幼体、清道夫、寄生者

战斗特点：正面硬打很难、可以诱敌离巢、可以堵入口、可以攻击资源链

---

# 10. 地图设计规则

每个区域至少包含：

```text
1 个主要资源
1 个主要威胁
1 条安全路线
1 条高收益危险路线
1 个可被利用的环境特征
1 个适合某种身体构筑的优势点
```

例如：

```text
酸池洞穴：
主要资源：酸晶
主要威胁：酸蚀虫
安全路线：绕行高台
危险路线：穿越酸池
环境特征：酸池可杀敌
优势构筑：酸抗脚垫 / 跳跃腿 / 远程诱饵
```

---

# 11. 生物生态位分类

先做 8 类：

## 11.1 小型猎物

速度快、血少、繁殖快、提供食物、怕噪音

作用：食物链底层、吸引捕食者、给玩家早期资源

## 11.2 清道夫

吃尸体、低攻击、嗅觉强、会跟随战斗现场

作用：让战斗产生后果、清理尸体、引导玩家理解气味系统

## 11.3 伏击捕食者

低移动、高爆发、喜欢暗处/洞口

作用：惩罚盲目探索、让感知器官有价值

## 11.4 追击捕食者

速度快、耐力高、开阔地强、窄洞弱

作用：制造逃跑压力、让地形选择有意义

## 11.5 重甲守卫

慢、防御高、守资源/巢穴、怕酸/电/背刺

作用：作为构筑检验、迫使玩家使用克制部件

## 11.6 群体虫群

单体弱、数量多、共享警报、怕范围攻击/窄通道

作用：测试 AoE、通道、队伍 AI

## 11.7 环境生物

植物、菌毯、寄生体、陷阱、发光体

作用：把地图变成生态系统，不只是敌人刷新点

## 11.8 Apex 生物

强、稀少、有领地、不一定必须击杀

作用：提供长期目标、塑造区域性恐惧

---

# 12. 碰撞架构

分层：

```text
Visual Mesh:
    好看即可，不参与精确碰撞

Physics Proxy:
    AABB / OBB / Sphere

Gameplay Volume:
    感知、伤害、危险、气味、声音

Navigation Graph:
    房间、门户、区域、路径成本
```

不让 visual mesh 决定玩法。

## 地形碰撞

```text
墙：AABB / OBB
地面：AABB tile
斜坡：OBB ramp
平台：AABB
洞口：Portal volume
危险区：AABB / Sphere trigger
资源点：Sphere
巢穴：Sphere + Portal
```

## 生物碰撞

```text
小型生物：1 Sphere
中型生物：Body OBB + Head Sphere
大型生物：Body OBB + limbs OBB/Sphere
蛇形生物：Chain of Spheres
飞行生物：Body Sphere + Wing OBB
```

## 攻击碰撞

```text
咬击：Sphere
爪击：OBB
冲撞：swept OBB
尾击：sphere chain
投射物：Sphere
范围攻击：Sphere trigger
```

足够做出大量行为，不需要 GJK。

---

# 13. 确定性物理约束

确定性物理下要避免：

```text
大量堆叠刚体
复杂摩擦
连续斜面滚动
物理驱动 ragdoll
高速小物体碰撞
细长物体穿透
复杂 concave mesh collider
不规则地形接触
浮点混乱的自由碰撞
```

更稳的做法：

```text
固定 Tick
离散状态
显式速度积分
简化接触解算
范围检测
路径图
动作状态机
确定性随机数
```

> 让物理服务玩法，不要让玩法依赖真实物理。

---

# 14. 推荐战斗结算方式

用：

```text
物理接触 + 行为状态 + 数据结算
```

而不是纯物理。

例如"咬击"：

```text
条件：
- 正在 BiteAction
- 目标在 MouthSphere 内
- 朝向角度满足
- 目标没有处于完全闪避状态

结算：
- 伤害 = 牙齿攻击力 - 目标护甲
- 追加：出血/抓取/恐惧
- 写入黑板：AttackerThreat += 20
```

既有空间判断，又不会被复杂物理拖死。

---

# 15. MVP 战斗规格

## 战斗核心

```text
发现
追逐
接触
攻击
受伤
逃跑
尸体
清道夫
```

## 攻击类型

```text
Bite
Charge
Projectile
Area Hazard
```

## 防御类型

```text
Armor
Speed
Stealth
Group
TerrainResistance
```

## 战斗目标

```text
杀死
驱赶
抢资源
拖延
逃脱
带回样本
```

---

# 16. MVP 地图规格

## 区域 1：实验场

目的：测试 AI、测试攻击、测试移动

地形：平地、少量障碍、一个资源点、一个小型敌人

## 区域 2：废弃温室

目的：食物链入门

地形：植物资源、小型猎物、清道夫、少量遮挡

机制：食物 → 小猎物 → 捕食者 → 尸体 → 清道夫

## 区域 3：酸性洞穴

目的：环境抗性和路径选择

地形：酸池、窄道、绕行路线、高收益资源

机制：酸抗部件、跳跃部件、诱敌进酸池

## 区域 4：机械废墟

目的：感知和伏击

地形：暗区、金属资源、巡逻机器人、噪音传播、狭窄通道

机制：听觉、夜视、潜行、诱饵

---

# 17. 推荐地图数据结构

```csharp
public sealed class HabitatZone
{
    public string Id;
    public OBB Bounds;

    public float Light;
    public float Moisture;
    public float Temperature;
    public float Danger;
    public float NoiseLevel;

    public TerrainTag TerrainTags;
    public ResourceTag ResourceTags;
    public HazardTag HazardTags;
}

public sealed class SpeciesSpawnProfile
{
    public string SpeciesId;

    public TerrainTag PreferredTerrain;
    public ResourceTag RequiredFood;
    public HazardTag AvoidHazards;

    public float MinLight;
    public float MaxLight;

    public int MinGroupSize;
    public int MaxGroupSize;

    public float TerritoryRadius;
    public float SpawnBudgetCost;
}

public sealed class DenPoint
{
    public string Id;
    public Vector3 Position;
    public float Radius;
    public SpeciesTag AllowedSpecies;
    public PortalRestriction Restriction;
}
```

生成逻辑：

```text
遍历 HabitatZone
↓
计算每个物种的 SpawnScore
↓
根据生态预算生成 Den
↓
从 Den 派生活动范围
↓
按资源、地形、危险度生成日常路线
```

---

# 18. 差异化定位

市面上已有：

```text
Ecosystem：自动进化生物
Rain World：程序化生态生存
Niche：基因适应
Spore：生物创造
Impossible Creatures：组合生物战斗
```

但少见的是：

```text
玩家设计身体
+
玩家设计 AI 蓝图
+
生物自主行动
+
地形和生态位强绑定
+
确定性系统可回放/可分析
```

核心差异：

> **不是我控制怪物战斗，而是我设计怪物的身体和脑子，让它在生态地图中证明自己。**

---

# 19. 克制建议

```text
不要追求真实物理
不要追求复杂 mesh 地形
不要追求精准格斗
不要一开始做大开放世界
```

应该做：

```text
Room-based 生态地图
Primitive collider 地形代理
Intent-based 战斗
Zone-based 物种分布
可解释 AI 行为
战后回放分析
```

第一版目标：

```text
一只玩家设计的生物
四种野生生物
三种地形生态
四种攻击方式
五种 AI 意图
一套战后决策日志
```

只要这个闭环好玩，方向就成立。
