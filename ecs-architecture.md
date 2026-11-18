# Ewil 活世界基座：ECS 架构设计

- 版本：v0.4（2026-11-18 用户确认扩大 MVP：村子扩到 12 个 NPC + 第二张地图遗迹）
- 日期：2026-11-18
- 目标产品：语音优先、轻量 2D、多人在线、实时 AI DM 的活世界 RPG（见 `Product_Concept_Ewil_ChaoChen__1.pdf`）
- 定位：先做基座，不做完整游戏。灵感来源是 Caves of Qud，但不是复刻 Qud。
- 技术基线：Rust + flecs_rust

> 一句话目标：基座要证明一件事——世界在没有玩家、没有 DM 讲话的时候也在自己运转，而且玩家做的事会留下可查询、可回放、不可悄悄抹掉的痕迹。

---

## 0. 已确认的技术基线

- 确认 1：语言用 Rust + flecs_rust API，服务端权威模拟，客户端只做展示和输入。手机 / PC 客户端不直接跑完整模拟。
- 确认 2：世界是 2D 网格、回合制。实时语音只是输入方式，底层模拟按 tick 推进，不按帧率推进。
- 确认 3：v1 有两张可玩地图：村子 joppa（60x45）+ 东边遗迹 red_sand_ruin（40x30）。不做“近 100 万张地图”。先证明闭环，再扩规模。
- 确认 4：AI DM 是“提议者 + 讲述者”，不是模拟器。所有世界状态改变必须经过模拟层校验和执行（见第 6 节）。

---

## 1. 设计目标与非目标

### 1.1 基座必须证明的 5 件事

1. **同构模拟**：玩家和 NPC 用同一套组件和系统。NPC 不是背景板，它也会饿、会走路、会记仇、会被打死。
2. **可查询的世界**：任何时刻都能回答“谁在哪、在干什么、和谁关系如何、这里发生过什么”，不需要 DM 靠猜。
3. **后果可持久**：一次攻击、一次交易、一次背叛都会进事件日志和关系表，存档重载后还在。
4. **数据驱动**：新生物、新物品、新变异靠改数据文件就能加，不用改 Rust 核心逻辑。
5. **DM 可接管但不可越权**：AI DM 能读世界、能提议行动、能要一次检定，但不能直接改血量、改位置、改声望。

### 1.2 v1 明确不做的事（非目标）

- 不做 3D、不做实时动作战斗、不做复杂物理（刚体、抛物线）。
- 不做完整液体模拟。v1 只做“地面上有水 / 火 / 毒雾”三类区域效果，不模拟流动、混合、蒸发。
- 不做 70 种变异、60 个派系。v1 只做 6 种变异、3 个派系（见第 5 节）。
- 不做程序化生成整张世界地图。v1 地图手工设计 + 少量程序化填充，历史生成只做“苏丹式”简化版。
- 不做经济全模拟。v1 只做固定价格 + 声望折扣，不做供需、通货膨胀。
- 不做 PvP 平衡和反作弊全套。v1 默认是合作房间，服务端校验只防非法命令，不防所有作弊。

### 1.3 成功的标准

- 一个新开发者照着本文能在 2 周内跑起最小闭环（见第 8 节），而不是只能跑起一个空 ECS 世界。
- 任何一次“世界为什么这样”的问题，都能用事件日志 + 组件快照回答，而不是用“AI 当时是这么说的”回答。

---

## 2. 核心 ECS 模型（Flecs）

### 2.1 建模原则

- 实体（Entity）只是一个 ID，没有行为。
- 数据只放在组件（Component）里，逻辑只放在系统（System）里。
- 关系用 Flecs 的 Pair 表示，不用在组件里塞 `faction_id`、`owner_id` 这种裸 ID 再到处查表。
- 生物和物品先定义 Prefab（模板），再实例化。数值、名字、描述一律来自数据文件，不写死在代码里。
- 组件要小、可序列化、可单独读 / 写。一个组件超过 8 个字段，先想想是不是该拆。

### 2.2 世界层级

```text
World（单例实体，挂 WorldState）
 └─ Map（地图实体，挂 MapInfo、Grid）
     ├─ Tile（v1 不给每个格子建实体，用数组 + 组件 TileEffect 存地面效果）
     ├─ Creature（玩家 / NPC，挂 Position 等）
     ├─ Item（挂 Position 或 OwnedBy）
     └─ Faction（派系实体，挂 FactionInfo）
```

- v1 有两个 `Map` 实体：`joppa`（60x45，村子）和 `red_sand_ruin`（40x30，东边遗迹）。每张地图一个 `MapGrid` 单例。格子数据存在 `MapGrid` 单例组件里，不给每个格子建实体，避免实体爆炸。
- 玩家离线时，他的实体保留在世界里，挂 `Offline` 标签，不参与回合，但可以被偷、被攻击（由规则决定，v1 默认离线玩家不可被攻击）。

### 2.3 核心组件表

| 组件 | 字段（Rust） | 说明 |
|---|---|---|
| `Position` | `x: i32, y: i32, map: Entity` | 网格坐标。移动只改这个组件，渲染只读它 |
| `Name` | `name: String, desc: String` | 展示名和一句话描述，供 DM 叙述用 |
| `Health` | `hp: i32, max_hp: i32` | 0 即死亡，死亡系统接管，不由战斗系统直接删实体 |
| `Stats` | `str_: i32, agi: i32, wil: i32, tou: i32, level: i32, xp: i32` | 基础四维 + 等级。检定用 `Stats` + 技能 + 变异修正 |
| `Body` | `max_parts: i32` | 身体总览。肢体是子实体，不是字段 |
| `BodyPart` | `slot: PartSlot, hp: i32, max_hp: i32, severed: bool` | 手、头、腿等。每个肢体是一个实体，用 `ChildOf` 挂在生物下 |
| `Faction` | `base_rep: i32` | 只挂在派系实体上。个人声望存在关系上，不存在这里 |
| `Reputation` | `value: i32` | 作为 Pair 的数据：`(MemberOf, 派系)` 的关系上可挂数据，或用单独的 `Reputation` 组件 + 目标字段。v1 用后者，见 2.4 |
| `Inventory` | `capacity: i32, weight: i32` | 容量上限。物品用 `OwnedBy` 关系挂在生物下，不用数组 |
| `Item` | `kind: ItemKind, weight: i32, value: i32` | 物品本体。`kind` 是枚举：武器 / 防具 / 消耗品 / 材料 / 任务品 |
| `Weapon` | `damage: i32, range: i32, type_: DamageType` | 只有武器实体挂 |
| `Mutation` | `mutation_id: String, level: i32` | 一个实体可挂多个。效果由 `MutationDef` 数据定义，见第 7 节 |
| `Skill` | `skill_id: String, level: i32` | 同上。v1 只做 6 个技能 |
| `Needs` | `hunger: i32, thirst: i32, fatigue: i32` | 0–100。超过阈值触发 AI 需求行为 |
| `Schedule` | `day_job: String, night_home: String, wake_tick: i32, sleep_tick: i32` | 白天去哪、晚上回哪。MVP 的核心组件 |
| `AIState` | `state: AIStateKind, target: Entity, state_tick: i32` | 当前状态：巡逻 / 工作 / 吃饭 / 睡觉 / 战斗 / 逃跑 / 跟随 |
| `Perception` | `sight_range: i32` | 视野范围。v1 只做曼哈顿距离 + 视线遮挡简化版 |
| `Memory` | 见 2.5 | 不是一个大组件，是一组关系 + 事件引用 |
| `Energy` | `energy: i32, speed: i32` | 回合制能量。见第 3 节 |
| `TurnActor` | （空组件 / 标签） | 能参与回合的实体才挂。墙、地面不挂 |
| `Player` | `account_id: String` | 玩家专属。NPC 不挂 |
| `Dead` | `death_tick: i32, cause: DeathCause` | 标签 + 死亡信息。尸体保留 100 tick 后清理 |
| `TileEffect` | `kind: EffectKind, intensity: i32, ttl: i32` | 地面效果：水、火、毒雾。存在 `MapGrid` 里，不单独建实体 |

标签（Tag，无字段）：`NPC`、`Hostile`、`Friendly`、`Merchant`、`QuestGiver`、`Offline`、`Invisible`、`Flying`。

### 2.4 关系（Pair）——这是 Flecs 最值得用的部分

Flecs 的 Pair 写法示例（flecs_rust API）：

```rust
// 肢体属于身体
arm.add((ChildOf, creature));

// 生物属于派系
creature.add((MemberOf, faction_apes));

// 物品属于某人
sword.add((OwnedBy, player));

// 装备在某个槽位
sword.add((Equipped, Slot::Hand));

// 生物位于某张地图（Position 里已有 map，关系用于查询）
creature.add((LocatedIn, map_joppa));
```

v1 必须用的关系：

| 关系 | 用法 | 查询示例 |
|---|---|---|
| `ChildOf` | 肢体 → 生物；也可以用于“村子里的房子属于村子” | 查一个生物的所有肢体 |
| `MemberOf` | 生物 → 派系 | 查某派系的所有成员 |
| `OwnedBy` | 物品 → 生物 | 查某人的背包 |
| `Equipped` | 物品 → 槽位（Pair 的目标是槽位实体或枚举） | 查某人手上拿了什么 |
| `LocatedIn` | 生物 / 物品 → 地图 | 查某张地图上的所有实体 |
| `HostileTo` | 生物 → 生物 / 派系 | 查“谁恨我” |
| `Knows` | 生物 → 生物 | 查“谁认识谁”，用于传话、告密 |

声望的存法（v1 定死，不要两种混用）：

```rust
struct Reputation {
    faction: Entity, // 派系实体
    value: i32,      // -100 ~ +100
    last_change_tick: i32,
}
```

- 每个生物实体上可以挂多个 `Reputation` 组件（Flecs 允许同一组件挂多个实例时要用 Pair 或换写法，v1 更稳妥的做法是：`Reputation` 挂在“生物—派系”关系实体上，或用一个 `Reputations` 组件内含 `[ReputationEntry; 8]`，上限 8 个派系）。
- 简化决策：v1 用 `Reputations { entries: [ReputationEntry; 8], count: usize }`，派系数量少，查询简单，序列化也简单。等派系超过 8 个再改成关系实体。

### 2.5 记忆（Memory）——不要做成黑盒

NPC 的“记得被打过”不能只存在 AI 的 prompt 里，必须是世界数据：

```rust
struct MemoryEntry {
    other: Entity,    // 和谁有关
    kind: MemoryKind, // 被攻击 / 被帮助 / 被偷 / 被救
    tick: i32,        // 什么时候
    strength: i32,    // 0–100，随时间衰减
    event_id: Entity, // 指向事件日志里的那条事件
}
struct Memories {
    entries: [MemoryEntry; 16],
    count: usize,
}
```

- 规则：`strength` 每 100 tick 减 1，低于 10 就遗忘（删除条目并记一条“遗忘”事件，可选）。
- AI 决策先读 `Memories`，再读 `Reputation`，最后才读当前 `Perception`。这样 NPC 的行为是可解释的。

### 2.6 Prefab 示例

Flecs Prefab 不是继承，是“复制一份再改”：

```rust
// 生物模板：村民
let villager = world.prefab_named("Villager")
    .set(Health { hp: 10, max_hp: 10 })
    .set(Stats { str_: 8, agi: 8, wil: 8, tou: 8, level: 1, xp: 0 })
    .set(Needs { hunger: 50, thirst: 50, fatigue: 20 })
    .add(NPC)
    .add(TurnActor)
    .add((MemberOf, faction_villagers));

// 实例化：名字和位置每次不同
let npc = world.entity()
    .is_a(villager)
    .set(Name { name: "阿木".into(), desc: "村里的铁匠，左手少了一根手指".into() })
    .set(Position { x: 12, y: 8, map });
```

物品模板同理。Prefab 本身不进模拟循环，实例化后才有 `Position` 和 `TurnActor`。

---

## 3. Tick / 模拟循环

### 3.1 基本决策

- 不用 Flecs 的帧循环驱动游戏。语音是实时的，模拟是回合的。
- 服务端维护一个 `WorldClock { int tick; int day; int hour; }` 单例。1 tick = 世界里 10 分钟（可调），1 天 = 144 tick。
- 每次 `world.progress()` 只跑一个逻辑 tick。不要用 `delta_time`，不要让帧率影响模拟。

### 3.2 能量制（Action Points）

每个 `TurnActor` 每回合获得能量：

```rust
struct Energy { energy: i32, speed: i32 } // speed 100 为基准
// 每回合：energy += speed
// 移动花费 100，攻击花费 100，等待花费 50，复杂动作花费 150–200
// energy >= 100 才能行动，行动后扣费；一回合内可多次行动，直到 energy < 100
```

- 玩家行动不立即结算，先进入命令队列。系统按 `speed` 排序，逐个执行。
- DM 讲话、玩家讨论不消耗 tick。只有“宣布行动并确认”才消耗 tick。这样语音的实时性和回合制的公平性可以共存。

### 3.3 一个 tick 的执行顺序（写死，不要靠系统注册顺序碰运气）

Flecs 的 Phase 要显式指定：

```text
Phase 1  PreUpdate   ：时钟推进、需求衰减（饿 / 渴 / 累 +1）、记忆衰减
Phase 2  OnUpdate    ：感知更新（谁看得见谁）
Phase 3  OnUpdate    ：AI 决策（只生成 Intent，不直接改世界）
Phase 4  OnUpdate    ：命令解析与执行（玩家命令 + AI Intent 统一进 Action 队列）
Phase 5  OnUpdate    ：战斗结算、移动结算、拾取 / 使用 / 交易
Phase 6  OnUpdate    ：派系与声望更新、事件日志写入
Phase 7  PostUpdate  ：死亡清理、尸体处理、地面效果衰减
Phase 8  PostFrame   ：快照生成、推送给客户端和 AI DM
```

关键规则：

- Phase 3 的 AI 只写 `Intent` 组件（`Intent { kind: ActionKind, target: Entity, x: i32, y: i32 }`），不直接改 `Position` / `Health`。Phase 4 再统一执行，这样回放和调试才有意义。
- 同一个 tick 内，两个实体想进同一个格子，按 `speed` 高者先行，低者原地等待并记一条“被挡住”事件，不许穿人。
- 所有随机数都走 `WorldRng { uint64_t seed; }`，不要用 `rand()` / 未播种的随机数。种子存在存档里，回放同一段事件日志应得到同一结果。

### 3.4 离线推进（Offline Progression）

- 没有玩家在线时，世界不停止，但降速：每 10 分钟真实时间推进 24 tick（相当于 4 小时游戏时间），且只模拟“已加载区域”。
- 远离玩家的区域进入抽象模式：不逐 tick 跑每个 NPC，只按 `Schedule` 跳状态（比如“铁匠白天在铺子，晚上回家”直接跳位置，不走路），并按概率生成摘要事件（“今晚村里丢了一只羊”）。
- 玩家上线时，抽象区域要能“展开”回具体状态：位置按 `Schedule` 推算，`Memories` 和 `Reputation` 保留，摘要事件已经写进日志，DM 可以直接讲出来。
- v1 两张地图都全量模拟（玩家在其中一张时，另一张按 `Schedule` 降速推进，不逐 tick 跑每个 NPC）。跨地图旅行是显式的出口移动，不做跨地图抽象，抽象是 v2 的事。

---

## 4. 系统列表（按优先级）

P0 是 MVP 必须有，P1 是基座完整后下一批，P2 是明确排后。

### P0

| 顺序 | 系统 | 输入 → 输出 | 备注 |
|---|---|---|---|
| 1 | `ClockSystem` | `WorldClock` → tick / day / hour | 只做时间，不做别的 |
| 2 | `NeedsSystem` | `Needs` → hunger / thirst / fatigue +1 | 阈值到 80 触发 AI 需求 |
| 3 | `PerceptionSystem` | `Position` + `Perception` → `Visible` 关系 | v1 用距离 + 简单遮挡，不做真实视线 |
| 4 | `AIScheduleSystem` | `Schedule` + `WorldClock` + `AIState` → `Intent` | 白天巡逻 / 工作，晚上回家 |
| 5 | `AICombatSystem` | `AIState` + `Memories` + `Reputation` → `Intent` | 被攻击过 → 逃跑或反击；声望低 → 拒绝交易 |
| 6 | `ActionSystem` | `Intent` / 玩家命令 → `Position` / `Health` / `Inventory` | 唯一能改核心状态的系统 |
| 7 | `CombatSystem` | 攻击 Intent → 检定 → `Health` / `BodyPart` | 检定公式见 4.1 |
| 8 | `MovementSystem` | 移动 Intent → `Position` | 碰撞、地形、负重都在这里判 |
| 9 | `DeathSystem` | `Health.hp <= 0` → `Dead` 标签 + 掉落 | 死亡不删实体，先变尸体 |
| 10 | `EventLogSystem` | 所有系统 → `EventLog` | 只追加，不修改（见第 5 节） |
| 11 | `SaveSystem` | 世界 → 存档文件 | Flecs 序列化 + 事件日志 |

### P1

| 系统 | 说明 |
|---|---|
| `FactionSystem` | 声望变化 → 派系态度 → NPC 行为阈值（见第 5 节） |
| `HistorySystem` | 开局生成苏丹式历史，生成地点、人物、仇恨链 |
| `TradeSystem` | 价格 = 基础价 × 声望折扣 × 稀缺系数，v1 不做供需 |
| `CraftSystem` | 配方表驱动：2 材料 + 工具 → 1 物品 |
| `TileEffectSystem` | 地面水 / 火 / 毒雾的伤害和持续时间 |
| `MutationSystem` | 变异生效：被动加成 + 主动技能，效果来自数据 |

### P2（v1 不做）

- 完整液体流动、温度、腐蚀、爆炸链。
- 跨地图旅行、世界地图生成、地下多层。
- 复杂经济、领地、建造。
- 多线程并行模拟。v1 单线程先跑通，Flecs 的多线程是优化，不是基座。

### 4.1 检定公式（v1 定死一个，别每个系统各搞一套）

```text
检定值 = d20 + 属性修正 + 技能等级 + 变异修正 + 环境修正
成功：检定值 >= 难度（DC，默认 12）
大成功：d20 == 20；大失败：d20 == 1
```

- 所有检定都走 `Dice::roll_check(attacker, defender, skill_id, dc)`，结果写进事件日志，DM 只负责讲，不负责算。
- 战斗伤害：`伤害 = 武器伤害 + 力量修正 - 防具`，最低 1。肢体命中先随机选一个未断肢体，伤害超过肢体血量则断肢。

---

## 5. 活世界：向 Qud 借什么、怎么借

### 5.1 向 Qud 借的 4 件事

1. **同构模拟**：Qud 里每个怪物和 NPC 都和玩家一样，有等级、技能、装备、派系、肢体。Ewil 也照做：不要给 NPC 写一套“假 AI”，要让 NPC 走同一套 `Health` / `Stats` / `Inventory` / `BodyPart`。
2. **派系即行为**：Qud 有 60 多个派系，声望高低直接决定 NPC 是帮忙还是攻击。Ewil v1 只做 3 个派系，但机制要先做对。
3. **历史生成任务**：Qud 每局生成一套苏丹历史，历史里的事件会变成遗迹、传说物品和任务。Ewil v1 做简化版：开局生成 3 位统治者、5 件大事、3 处遗迹，任务从这里派生。
4. **世界可破坏、可留痕**：Qud 里墙可以挖、可以熔。Ewil v1 不做全破坏，但要做到“门被烧了就是烧了，村里人都知道”。

### 5.2 派系与声望（v1：3 个派系）

| 派系 | 定位 | 初始态度 |
|---|---|---|
| `村民`（Villagers） | 村子居民，铁匠、商人、农夫 | 中立偏友好 |
| `拾荒者`（Scavengers） | 流浪者，偶尔交易，偶尔抢劫 | 中立偏敌对 |
| `机械教团`（Mechanists） | 崇拜旧机器的团体，守着一处遗迹 | 冷漠，只认“献上零件” |

声望规则：

```text
-100 ~ -50：敌对，见面即攻击，拒绝交易
 -49 ~  -1：冷淡，价格 ×1.5，拒绝任务
   0 ~  49：中立，正常价格
  50 ~ 100：友好，价格 ×0.8，会主动帮忙、给情报
```

- 声望变化只通过事件：完成任务 +10，帮助 +5，攻击 -20，偷窃被发现 -15，水之仪式（向传奇生物献水）+15。
- 声望是双向的：你帮村民打拾荒者，村民 +10，拾荒者 -10。不要只加不减。
- 派系态度存在派系实体上，个人声望存在生物的 `Reputations` 里。AI 决策时先看个人声望，再看派系平均声望。

### 5.3 历史生成（Sultan 式，简化版）

开局只做一次，生成后写进 `History` 单例和事件日志：

```text
输入：世界种子
输出：
  - 3 位统治者：名字、称号、在位时间、死因
  - 5 件大事：战争 / 瘟疫 / 丰收 / 背叛 / 发现遗迹
  - 3 处遗迹：位置、属于哪位统治者、里面有什么
  - 2 条仇恨链：A 派系恨 B 派系，因为某件大事
```

- 每件大事要落到具体东西上：不是“曾经有一场战争”，而是“三十年前红沙之战，铁匠的父亲死在东边遗迹，村里从此不欢迎拾荒者”。这样 DM 才有可讲的、玩家才有可查的。
- 传说物品：每个遗迹放 1 件有名字的物品，物品的 `Name.desc` 里写一句话历史。玩家捡到就能直接讲故事。

### 5.4 事件日志（Event Log）——活世界的证据

这是基座最重要的单例之一：

```rust
struct WorldEvent {
    event_id: u64,
    tick: i32,
    kind: EventKind,          // 移动 / 攻击 / 交易 / 死亡 / 对话 / 发现
    actor: Entity,     // 谁做的
    target: Entity,    // 对谁 / 对什么
    summary: String,     // 一句话，供 DM 和玩家查询
    data_json: String,   // 结构化数据，供回放和结算
};
struct EventLog {
    events: Vec<WorldEvent>, // 只追加，永不修改、永不删除（v1 上限 10000 条，超了归档到文件）
};
```

- 规则：任何改变世界状态的系统都必须写一条事件。没有事件，就等于没发生。
- 玩家行动的痕迹：烧了门 → 事件“烧门” + 地图上那扇门实体挂 `Burned` 标签 + 村民 `Memories` 里加一条。DM 不需要记住，查日志就行。
- 回放：按 `event_id` 顺序重放，就能还原世界。存档 = 组件快照 + 事件日志，两者都要有。

---

## 6. AI DM 接口：DM 能做什么、不能做什么

### 6.1 接口边界（一句话）

**DM 只能读世界、提议行动、要一次检定；不能直接写组件。**

所有写操作必须走 `ActionCommand`，由模拟层校验、结算、写日志。

### 6.2 DM 能读的（只读查询）

| 查询 | 返回 | 用途 |
|---|---|---|
| `get_snapshot(map, radius)` | 指定范围内的实体、位置、血量、状态 | DM 描述场景 |
| `get_entity(entity_id)` | 名字、描述、组件摘要、声望、记忆 | DM 介绍 NPC |
| `query_events(since_tick, kind)` | 事件列表 | DM 讲“刚才发生了什么” |
| `query_faction(faction_id)` | 派系态度、成员、仇恨链 | DM 讲政治 |
| `query_history()` | 统治者、大事、遗迹 | DM 讲传说 |
| `get_visible(player_id)` | 玩家看得见的实体 | 防止 DM 剧透看不见的东西 |

- 快照是“DM 视角”的，不是全图。玩家看不见的，DM 也不应该直接讲出来（除非是全知叙述，v1 默认不允许）。
- 快照格式用 JSON，字段固定，不要让 DM 自己拼 Flecs 查询。

快照示例（结构，不含真实数值）：

```json
{
  "tick": 1234,
  "day": 3,
  "hour": 14,
  "location": {"map": "joppa", "x": 12, "y": 8},
  "visible": [
    {"id": "npc_12", "name": "阿木", "kind": "villager", "hp": 10, "state": "working"}
  ],
  "recent_events": [
    {"tick": 1230, "summary": "阿木回到铁匠铺开始工作"}
  ]
}
```

### 6.3 DM 能提议的（ActionCommand）

DM 把自然语言翻译成结构化命令，再交给模拟层：

```rust
enum class ActionKind {
    Move, Attack, Talk, Trade, PickUp, Drop, UseItem,
    Sneak, PickLock, Craft, Rest, Wait, GiveItem, Ritual
};
struct ActionCommand {
    actor: Entity,
    kind: ActionKind,
    target: Option<Entity>, // 可为空
    x: i32, y: i32,             // 目标位置，可为空
    skill_id: String, // 检定用的技能，可为空
    dc: i32,               // 难度，默认 12
    raw_text: String, // 玩家原话，存档用，不参与结算
};
```

- 流程：玩家说话 → DM 生成 1 个或多个 `ActionCommand` → 模拟层校验（距离够吗？有这个物品吗？能量够吗？）→ 校验失败返回原因给 DM，DM 用自然语言解释 → 校验通过则结算 → 写事件 → 快照更新 → DM 讲述结果。
- DM 一次只能提议，不能连续提议 3 个动作绕过能量限制。能量不够就是不够。

### 6.4 DM 绝对不能做的事

- 不能直接改 `Health`、`Position`、`Reputation`。必须走 `ActionCommand`。
- 不能凭空创造物品、NPC、金钱。创造也要走 `SpawnCommand`，且要有来源（事件、任务、历史）。
- 不能改事件日志。日志只追加。
- 不能读其他房间的世界数据。每个房间一个 `World` 实例，隔离。
- 不能在玩家看不见的地方替玩家做决定。DM 可以建议，不可以代打。

### 6.5 检定与叙述的分工

| 事 | 谁做 |
|---|---|
| 骰子、命中、伤害、声望变化 | 模拟层（Rust） |
| 把结果讲成故事、描述场景、扮演 NPC 说话 | AI DM |
| 玩家想做规则之外的事（比如“用影子藏起来”） | DM 翻译成最接近的 `ActionCommand`（Sneak + 环境修正），模拟层判成功与否 |
| 规则没覆盖的事 | DM 标记为 `Unruled`，模拟层记一条事件，事后补规则，不许当场编数值 |

---

## 7. 数据驱动内容

### 7.1 原则（v0.3 已确认）

- 生物、物品、变异、技能、派系、配方、历史模板全部是数据文件，Rust 只认 ID。
- 主格式是 JSON，一个实体一个文件，每类一个目录：`data/creatures/`、`data/items/`、`data/mutations/`、`data/factions/`、`data/recipes/`。例如 `data/creatures/villager_blacksmith.json` 只定义铁匠一个实体。
- TOML 是可选的作者格式，不是第二套模型（见 7.6）。人手写、社区投稿可以用 TOML，运行时和 AI 生成用 JSON，两者落到同一套 Rust struct。
- 数据文件以后也要能由 AI DM 和玩家社区来写，所以必须有模板、Schema、校验工具和例子，不能只靠口头约定（见 7.7）。
- 加载时校验：ID 唯一、引用存在、数值在合理范围。校验失败要报错并拒绝启动，不要静默用默认值。
- 每个数据文件带 `version` 字段，当前为 1。加载器用 serde 的 `deny_unknown_fields` 严格校验，未知字段直接报错。

### 7.2 生物数据示例

```json
{
  "id": "villager_blacksmith",
  "name": "铁匠",
  "desc": "村里的铁匠，左手少了一根手指",
  "prefab": "Villager",
  "stats": {"str": 12, "agi": 8, "wil": 10, "tou": 12},
  "health": 14,
  "faction": "villagers",
  "skills": [{"id": "smithing", "level": 3}],
  "schedule": {"day": "work_at_forge", "night": "go_home"},
  "mutations": []
}
```

### 7.3 物品数据示例

```json
{
  "id": "iron_sword",
  "name": "铁剑",
  "desc": "一把普通的铁剑，刃口有几个缺口",
  "kind": "weapon",
  "weight": 3,
  "value": 20,
  "weapon": {"damage": 6, "range": 1, "type": "slash"}
}
```

生物文件里的 `mutations` 是 `[{id, level}]` 列表，和 `skills` 同一种写法，空列表表示没有变异。

### 7.4 变异数据示例（v1 只做 6 种）

v1 变异清单：`夜视`、`利爪`、`厚皮`、`心灵感应（弱）`、`喷酸（弱）`、`多臂（只能拿多一件武器）`。每种都要写清是被动还是主动、消耗什么、冷却多久。

```json
{
  "id": "night_vision",
  "name": "夜视",
  "kind": "passive",
  "effect": {"perception_bonus_night": 3},
  "desc": "在黑暗中看得更远"
}
```

### 7.5 派系数据示例

```json
{
  "id": "villagers",
  "name": "村民",
  "desc": "住在村子里的普通人",
  "home_map": "joppa",
  "attitude": {"scavengers": -20, "mechanists": 0}
}
```

### 7.6 TOML 兼容：JSON 为主、TOML 可选

- 设计原则：文件格式不决定数据模型。Rust 里先定一套 canonical struct，JSON 和 TOML 都用 serde 反序列化到同一套 struct，校验、ID 唯一、引用检查只写一次。
- 加载器按后缀分流：`.json` 走 serde_json，`.toml` 走 serde_toml，读进来之后完全一样。不要为两种格式各写一套加载逻辑。
- 同一个 id 不许在 JSON 和 TOML 里各出现一次。加载时跨格式查重，重复就报错并拒绝启动。
- 同一个实体不要同时保留 `.json` 和 `.toml` 两个文件。选一种格式存，其余用转换工具生成，不提交两份。

交集规则（两种格式都能写的子集，超出的一律不许用）：

- 不用 `null`。可选字段用“缺这个 key”表示，对应 Rust 的 `Option<T>`。
- 枚举都用字符串，不用数字代号。
- 不用混合类型数组，数组里所有元素同一种结构。
- 嵌套不超过 3 层。超过的拆成两个实体，用 id 引用。
- 数字统一用整数，浮点只在明确需要的地方用，且写清单位。
- 列表在 TOML 里用 array of tables 写，在 JSON 里就是普通数组，结构必须一一对应。

TOML 同一实体的写法示例（与 7.2 的铁匠 JSON 等价）：

```toml
id = "villager_blacksmith"
name = "铁匠"
desc = "村里的铁匠，左手少了一根手指"
prefab = "Villager"
health = 14
faction = "villagers"

[stats]
str = 12
agi = 8
wil = 10
tou = 12

[[skills]]
id = "smithing"
level = 3

[schedule]
day = "work_at_forge"
night = "go_home"
```

### 7.7 给 AI DM 和社区写数据的支持

- 每个类型给一个模板文件：`data/templates/creature.json`、`data/templates/item.json` 等，复制改名就能用。
- 每个类型给一个例子：JSON 和 TOML 各一份，内容等价，放在 `data/examples/` 里，测试要保证两者解析结果一致。
- 提供校验工具：`ewil-data validate data/`，检查 ID 唯一、引用存在、数值范围、未知字段，失败时报文件名、行号和具体原因。
- 提供转换工具：`ewil-data convert <in> <out>`，TOML 和 JSON 互相转。审稿看 JSON，动手改用 TOML，提交前转回 JSON 也可以。
- AI DM 生成数据时只许输出 JSON，输出后必须过 `validate`，不通过就退回重写，不许直接进世界。
- 社区投稿的流程：先交 TOML 或 JSON，`validate` 通过 + 测试解析一致，才合并。合并后由维护者决定 canonical 文件用哪种格式存。

---

---

## 8. MVP 里程碑：最小闭环（v0.4 扩大版）

### 8.1 闭环描述

**“一个村子记得你，一处遗迹等着你。”**

场景：

- 地图 1 `joppa`：60x45，一座村子（8 间房子、1 个铁匠铺、1 间旅店、1 间诊所、1 口井、1 个集市摊位、村口），村外一条路向东通往遗迹。
- 地图 2 `red_sand_ruin`：40x30，东边遗迹（破损大厅、机器祭坛、1 件传说物品红沙权杖）。遗迹是第 5.3 节历史生成的落点：三十年前红沙之战的战场，村里从此不欢迎拾荒者。
- 角色：1 名玩家、16 个生物（12 名村民 + 村外 3 名拾荒者 + 遗迹里 1 名机械教团隐士），3 个派系全部在跑，6 种变异至少 3 个 NPC 真正带着，物品 10 件。
- 时间：从第 1 天早上 8 点开始，玩家可以自由行动 5 个游戏日（720 tick）。

生物名册（数据在 `data/creatures/`，一个实体一个文件）：

| 生物 | 地图 | 派系 | 变异 |
|---|---|---|---|
| 铁匠阿木 `villager_blacksmith` | joppa | villagers | 无 |
| 商人阿珍 `villager_merchant` | joppa | villagers | 无 |
| 巡逻兵铁柱 `patrol_guard_1` | joppa | villagers | 无 |
| 巡逻兵石头 `patrol_guard_2` | joppa | villagers | 厚皮 |
| 农夫老田 `villager_farmer` | joppa | villagers | 无 |
| 医生白芷 `villager_doctor` | joppa | villagers | 无 |
| 旅店老板娘阿桂 `villager_innkeeper` | joppa | villagers | 无 |
| 村长石爷 `villager_elder` | joppa | villagers | 无 |
| 学徒小锤 `villager_apprentice` | joppa | villagers | 多臂 |
| 厨子阿香 `villager_cook` | joppa | villagers | 无 |
| 猎手夜鹰 `villager_hunter` | joppa | villagers | 夜视 |
| 孩子小豆 `villager_child` | joppa | villagers | 无 |
| 拾荒者瘦猴 `scavenger_1` | joppa（村外） | scavengers | 无 |
| 拾荒者爪子 `scavenger_2` | joppa（村外） | scavengers | 利爪 |
| 拾荒者酸水 `scavenger_3` | joppa（村外） | scavengers | 喷酸（弱） |
| 隐士锈眼 `mechanist_hermit` | red_sand_ruin | mechanists | 心灵感应（弱） |

物品（数据在 `data/items/`，共 10 件）：铁剑、铁矛、皮甲、木盾、伤药膏、水囊、烤面包、开锁器、机器零件在村子里或拾荒者身上，红沙权杖只在遗迹里。派系和变异数据在 `data/factions/`（3 个）和 `data/mutations/`（6 个）。

初始放置不在生物文件里：生物文件没有地图字段，放置由本节场景表决定。村民出生在 joppa，拾荒者出生在 joppa 村外，隐士出生在 red_sand_ruin。TOML 不为每个新文件各建一份，只保留 `data/examples/villager_blacksmith.toml` 作为 JSON/TOML 等价证明，其余文件用转换工具生成。

### 8.2 必须跑通的 8 个行为

1. 日程：铁匠白天在铺子工作，晚上 20 点回家睡觉；巡逻兵白天沿村子巡逻，晚上回村口站岗；猎手白天去村外打猎，晚上回家；隐士整天守在遗迹里，白天守卫、晚上冥想。位置变化可查。
2. 记忆与声望：玩家攻击铁匠一次，铁匠 `Memories` 里多一条“被玩家攻击”，声望 -20，之后拒绝交易并喊巡逻兵。
3. 派系连锁：玩家帮村民打跑拾荒者，村民声望 +10，拾荒者声望 -10，商人价格下降；玩家把机器零件献给隐士，机械教团声望 +15，隐士告诉玩家权杖的位置。
4. 跨地图旅行：玩家从 joppa 村东出口走到 red_sand_ruin，`Position.map` 变化、快照更新、状态不丢；再走回来也一样。隐士在玩家离开又回来之后还记得玩家。
5. 遗迹与传说物品：玩家在遗迹里找到红沙权杖，事件日志记一条“发现红沙权杖”；拿着权杖回村，村长石爷认出它并讲出红沙之战的故事。声望不够时隐士会阻止玩家拿走权杖。
6. 变异在起作用：夜鹰夜里视野更远，石头受击伤害更低，爪子的徒手伤害更高。至少 3 个带变异的 NPC 要在检定和战斗里表现出不同。
7. 交易随声望变：商人价格按第 5.2 节的声望档位变化，声望低时拒绝交易和任务。
8. 玩家第二天、第五天再去，铁匠还记得（存档重载后也记得），遗迹里的权杖状态也记得。

### 8.3 验收标准（全部要能演示，不是口头说）

- [ ] 启动后 10 秒内进入村子，玩家能移动、查看、对话。
- [ ] 不操作 50 tick，NPC 位置按 `Schedule` 变化，事件日志里有对应记录。
- [ ] 攻击 NPC 后，查 `Memories` 和 `Reputations` 能看到变化，DM 能讲出“他还记得你打过他”。
- [ ] 跨地图：从 joppa 走到 red_sand_ruin 再走回来，NPC 位置、记忆、声望、事件日志都在，隐士还记得玩家。
- [ ] 遗迹：找到红沙权杖，事件日志有记录；回村给村长看，村长讲出红沙之战。
- [ ] 派系连锁：帮村民打跑拾荒者后，村民声望升、拾荒者声望降、商人价格降；献机器零件后，机械教团声望升。
- [ ] 存档、退出、重载（跨地图旅行后也测一次），NPC 位置、记忆、声望、事件日志、权杖归属都在。
- [ ] 同一套代码跑 3600 tick 不崩溃、不 panic，实体数稳定（`cargo test` 通过）。
- [ ] 新加一个生物只需加一个 JSON 文件（或等价的 TOML 文件）+ 一行注册，不改核心系统。

### 8.4 验收用的测试脚本（建议先写，再写系统）

1. `test_schedule`：快进 144 tick，断言铁匠白天在铺子、晚上在家，巡逻兵白天巡逻、晚上在村口。
2. `test_memory`：攻击一次，快进 10 tick，断言 `Memories.count == 1` 且 `Reputation == -20`。
3. `test_persistence`：存档 → 重载 → 断言记忆和声望不变。
4. `test_replay`：同一段事件日志重放两次，断言最终世界状态一致。
5. `test_cross_map`：从 joppa 走到 red_sand_ruin 再走回来，断言 `Position.map` 变化两次、隐士记忆还在、事件日志有两次跨地图事件。
6. `test_faction_chain`：帮村民打跑拾荒者，断言村民声望 +10、拾荒者声望 -10、商人价格下降；献机器零件，断言机械教团声望 +15。
7. `test_ruin`：拿起红沙权杖，断言事件日志有记录、权杖归属是玩家；回村见村长，断言村长讲出红沙之战（事件或对话记录可查）。
8. `test_mutation`：夜里断言夜鹰的视野比普通村民远；断言石头的受击伤害比同条件下的铁柱低。

### 8.5 MVP 之后先做什么（按顺序）

1. 历史生成完整版：3 位统治者 → 5 件大事 → 3 处遗迹（现在只有东边遗迹可玩，另外 2 处先只做传说和任务钩子）。
2. 液体 / 地面效果完整版：火会蔓延、水会灭火。
3. 制作 + 交易完整版。
4. AI DM 接入：先接只读查询 + `ActionCommand`，再接叙述。
5. 多人房间：先做 2–4 人同房间共享同一个 `World`，再做离线推进。
6. 跨地图抽象：玩家不在的地图按 `Schedule` 跳状态，不逐 tick 跑每个 NPC。


---

## 附录 A：建议的目录结构

```text
ewil/
  src/
    main.rs                  // 启动、加载数据、跑 tick
    world/                   // World、Clock、Rng、EventLog
    components/              // 所有组件定义
    systems/                 // 所有系统，按第 4 节分文件
    data/                    // JSON 加载、校验
    dm/                      // AI DM 接口：快照、查询、ActionCommand
    save/                    // 存档、读档、回放
  data/
    creatures/ items/ mutations/ factions/ recipes/
  tests/
    test_schedule.rs
    test_memory.rs
    test_persistence.rs
    test_replay.rs
```

## 附录 B：命名约定

- 组件用 PascalCase：`Position`、`Health`、`Reputation`。
- 系统用 PascalCase + `System`：`MovementSystem`、`FactionSystem`。
- 数据 ID 用 snake_case：`villager_blacksmith`、`iron_sword`、`night_vision`。
- 事件 kind 用 snake_case：`attack`、`trade`、`death`、`discover`。
- 不要用缩写当 ID，`hp` 可以，`dmg` 不要，写 `damage`。

## 附录 C：主要风险

- **风险 1：系统顺序靠注册顺序。** 对策：第 3.3 节的 Phase 写死，写测试断言执行顺序。
- **风险 2：DM 绕过模拟。** 对策：第 6 节的接口是唯一写入口，代码评审时任何直接改 `Health` 组件都要被拦下。
- **风险 3：记忆无限增长。** 对策：`Memories` 上限 16 条，`EventLog` 上限 10000 条，超了归档。
- **风险 4：一开始就做太大。** 对策：第 8 节的 MVP 是上限，不是下限。跑不通就砍内容，不砍基座。
