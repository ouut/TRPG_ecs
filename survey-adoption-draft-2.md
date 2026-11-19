# Ewil 调研采纳草案 2：对象广告分 + 物质属性表

- 日期：2026-11-18
- 状态：**讨论草案，未采纳，未写进 `ecs-architecture.md`**。只有用户逐条定下来之后，才把采纳的部分写进架构文档。
- 来源：`game-survey.md` 采纳排序 Top 2（对象广告分效用决策，Sims 1）与 Top 3（物质属性表 + 地块表面状态，Qud、BG3、DOS2、Minecraft、DF），对应跨游戏模式 P8、P5、P6。
- 范围：只做这两条；Top 1 + Top 4 在 `survey-adoption-draft.md`，Top 5（秘密/谣言）及以后另议。
- 所有数值都是**第一版草案值**，跑模拟时再调。

---

## 1. 现状对照（以 `ecs-architecture.md` v2.0 为准，2026-11-18 实测）

| 现在已有的 | 位置 | 缺口 |
|---|---|---|
| `get_available_actions`：一个函数，两处共用（`DecisionSystem` 枚举 NPC 候选、DM 查询同一份列表） | §5.0 | 候选列表是按**硬编码规则**枚举的：位置、视野内实体、背包、附近地块、技能、日程/需求、能量。规则里写死了“食物→饿、 水→渴”这种映射；对象自己不宣告能满足什么需求 |
| 效用打分六项权重，`need` 项 = 该行动缓解的最急需求 / 100 | §5.0.1、`data/ai/utility_weights.json` | `need` 项不知道“这个对象能缓解多少”——吃面包缓解多少饿，是写在物品 `consumable.hunger_restore: 25` 里的，`get_available_actions` 枚举时没把这个数变成候选动作的广告分 |
| 物品 10 件：`iron_sword` 伤害 6、`iron_spear` 伤害 5、`wooden_shield` 防御 1、`leather_armor` 防御 2，全部手写数字 | `data/items/*.json` | **没有任何一个物品带 `material` 字段**。铁剑和木剑的区别只是一个伤害数字；“铁克木、火克木”这种交互定律无从推导，只能逐件手写 |
| `kind: "material"` 只是物品种类枚举（`lockpick`、`machine_part` 是材料类物品），不是材质 | `data/items/lockpick.json`、`machine_part.json` | 别把“材料类物品”和“材质属性”混在一起：前者是能捡的东西，后者是铁/木/皮是什么 |
| 地块 `Plot`：`Soil` / `Moisture` / `Temperature` / `Vegetation` / `MicrobeDensity`，唯一写者 `EnvironmentSystem` | §2.2、§4.3 | 没有**表面状态**：湿、油、血、燃这些“地块表面沾了什么”没地方存 |
| 地面效果 `TileEffect { kind, intensity, ttl }`：水、火、毒雾，唯一写者 `TileEffectSystem` | §2.3、§4.3 | `TileEffect` 是有时限的效果（ttl 减 1、蔓延、发伤害事件），不是持久的表面状态；血迹、油渍这种“留下来、慢慢变质”的东西没有归宿 |

---

## 2. 草案 C：对象广告分（Top 2）

### 2.1 想法

Sims 1 的做法（`game-survey.md` §3，有据）：每个可互动对象带“广告分”，向周围 Sim 宣告自己能满足何种需求、多少分；Sim 扫描周围对象的广告分 × 当前需求缺口 × 人格倍率，选最高分行动。Ewil 的映射：**广告分就是 `get_available_actions` 枚举时给每个候选动作贴的分数标签**，决策打分时直接用它，不再由 `DecisionSystem` 硬编码“面包能吃”。

```text
candidate_score(need) = 广告分(对象, 需求) × 需求缺口 / 100 × 人格乘数
```

- 广告分写在数据里，不写死在代码里：物品、NPC、地块都可以带广告。
- 人格乘数复用 §5.1 的亲和度：整洁高的人看到脏盘子（清洁广告）得分放大，邋遢的人近乎无感——Sims 的原话，Ewil 的 48 面可以直接当乘数表。
- `get_available_actions` 的两处调用方不变：NPC 和 DM 还是同一份列表；变的只是列表里的每个候选多了一个“为什么值得做”的分数来源。

### 2.2 用现有数据验算（全部取自 `data/` 真实值）

| 对象 | 现在的数据 | 按草案会宣告什么 |
|---|---|---|
| `bread_loaf` 面包 | `consumable.hunger_restore: 25`，`weight: 1`，`value: 5` | `hunger: 25`——饿 90 时吃面包的 need 项是 0.9 × 25 的量级，不再是“能吃”这个布尔值 |
| `water_flask` 水壶 | `consumable.thirst_restore: 30` | `thirst: 30`；顺带一个 Qud 式问题：水在 Ewil 里只是消耗品，还是同时是货币/外交筹码（见 §4 第 4 问） |
| `healing_salve` 药膏 | `consumable.heal_amount: 6` | `health: 6`——但 `Needs` 里没有“伤”这一项，药膏缓解的不是需求，是 `Health`；广告分要不要覆盖非需求类目标（回血、回能量），见 §4 第 2 问 |
| 铁匠 / 商人等 NPC | 视野内可对话 → `Talk`、可交易 → `Trade`（§5.0 现有规则） | `social: ?`——谈话缓解多少社交需求，现在没有数；草案 B 的 `social` 需求（`survey-adoption-draft.md` §3.2）如果采纳，谈话对象要带 `social` 广告分，否则需求进了候选却没有可缓解的对象 |
| 地块（可挖/可种/可烧） | `Dig` / `Plant` / `Burn` 由 §5.0 规则枚举 | 地块宣告的是**材料产出**不是需求：挖土得土块、烧草得灰——广告分机制只覆盖“满足需求”的对象，产出类对象走另一条（工作/制作打分），不要硬塞进同一张表 |

### 2.3 一个诚实的限制：现在能打广告的对象很少

当前世界只有 10 件物品、16 个生物、两张地图的地块，**没有任何床、椅子、火堆、桌子这类家具/设施实体**（`data/` 与文档全文检索无 `bed` / `furniture`）。Sims 的广告分之所以强，是因为满屋子对象都在打广告；Ewil 现在能打广告的主要是 3 件消耗品 + NPC 谈话/交易 + 地块产出。

所以草案 C 的真实收益分两档：

- **现在就能拿到的**：消耗品的广告分（面包、水、药膏）+ 谈话/交易的广告分（配合草案 B 的新需求），把 §5.0 里“食物→饿”这种硬编码映射收进数据。
- **要等对象多起来才有的**：床（休息广告）、火堆（取暖/烹饪广告）、工作台（制作广告）——这些对象实体本身还没设计，广告分机制先搭架子，等对象进来再填表。

### 2.4 与草案 B 的接口

- 草案 B（需求多样化）如果采纳，新增需求（`social` / `solitude` / `craft` / `thrill` / `order` / `romance`）的缓解对象**必须**有对应的广告分，否则需求进了效用打分却永远找不到缓解动作。
- 反过来，草案 C 如果先做、草案 B 后加，广告分表一开始只有 3 项生存需求 + 回血，表是空的——先做 C 不亏，但别指望它立刻让 NPC 行为变丰富。

---

## 3. 草案 D：物质属性表 + 地块表面状态（Top 3）

### 3.1 新增组件与数据表

```rust
// （3）物质与物理类。Material 的唯一写者：没有系统写它，它是静态属性表，
// 由 CombatSystem / CraftSystem / TileEffectSystem / EnvironmentSystem 读。
struct Material {
    material_id: String,   // 如 iron / wood / leather / flesh / stone / soil
    hardness: i32,         // 0-10：硬度，决定武器破甲与护甲减伤
    flammability: i32,     // 0-10：易燃，决定火蔓延速度
    density: i32,          // 0-10：密度/重量感，决定负重与钝击
}
```

- 数据表：`data/materials/<id>.json`，开放注册表（与技能注册表同构：加一个文件就是加一种物质）。
- 物品挂 `material_id`：`iron_sword` → `iron`，`wooden_shield` → `wood`，`leather_armor` → `leather`，`bread_loaf` → `organic`，`machine_part` → `iron`（或 `steel`，见 §4）。
- **交互定律由此表推导，不逐条手写**（Qud 的做法，`game-survey.md` §9）：

```text
武器伤害 = 基础伤害 × (武器材质 hardness / 目标护甲材质 hardness) 的档位修正
火蔓延：TileEffectSystem 每 tick 按 flammability 查表决定向邻格蔓延的概率
  （wood 8、leather 5、iron 0、soil 0、organic 6，草案值）
负重：物品重量 = 体积 × density，不再每个物品手写 weight
```

### 3.2 两档粒度（直接回答 survey 开放问题 2）

| 档 | 字段 | 代价 | 收益 |
|---|---|---|---|
| **Tier 1（建议先做）** | `hardness`、`flammability`、`density` 三项 | 10 件物品 + 几种地块材质（土/石/木/草）填表，一下午 | 武器/护甲不再是手写数字对比；火蔓延有依据；铁克木、火克木从表里算出来 |
| **Tier 2（后加）** | 再加 `temperature`（当前温度）、`melting_point`、`boiling_point`、`conductivity`（导热） | 每种物质多 4 个字段；`Temperature` 要进 tick 更新，`EnvironmentSystem` 的工作量变大 | 完整相变（水结冰/蒸发、金属熔化）、导热传导；但这是 Qud/ONI 级的模拟，超出基座范围（survey 跳过清单已写明不做完整液体模拟） |

草案建议：**Tier 1 现在做，Tier 2 明确后加**。Tier 1 的三项已经能覆盖当前世界的全部交互（打、烧、搬）；温度/相变等第一轮模拟跑完、真有玩法需要时再加。

### 3.3 地块表面状态：放哪？

Top 3 的后半句是“Plot 加表面状态（湿/油/血/燃）”。现状是 `TileEffect` 管有时限的效果、`Plot` 管土壤/湿度/温度/植被，表面状态两边都能放，草案建议：

```rust
// （8）世界与环境类，挂在 Plot 实体上。唯一写者 EnvironmentSystem。
struct Surface {
    kind: SurfaceKind,  // Wet / Oily / Bloody / Burning / Ashen
    intensity: i32,     // 0-100
    ttl: i32,           // 剩余 tick；血迹慢慢干、油渍慢慢挥发，到 0 自动清除
}
```

- 为什么不放 `TileEffect` 里：`TileEffect` 是“正在发生的效果”（火在烧、水在淹），`Surface` 是“留下的痕迹”（烧完的灰、干掉的血）。两者生命周期不同：效果 ttl 短、会发伤害事件；表面 ttl 长、只影响后续交互（血迹引来野兽、油面一点就着、湿地减速）。
- 与草案 A 的接口：如果草案 A（Thought→Emotion→Stress）采纳，`Bloody` 表面是“目睹尸体/血迹”想法的天然来源；如果草案 A 后加，`Surface` 也可以先只做物理交互（湿地减速、油面助燃），不产生想法。

### 3.4 用现有物品验算 Tier 1（草案值）

| 物品 | 现在的手写数 | 按 Tier 1 表推导 |
|---|---|---|
| `iron_sword`（铁） | 伤害 6 | `iron: hardness 8`；打 `leather_armor`（`leather: hardness 3`）全额，打铁甲则档位削减——现在没有铁甲，这个区分暂时用不上，但表先立住 |
| `wooden_shield`（木） | 防御 1 | `wood: hardness 3, flammability 8`；挡铁剑减伤少，还怕火——现在 `defense: 1` 这个数其实就是 `hardness 3` 的另一种写法 |
| `leather_armor`（皮） | 防御 2 | `leather: hardness 3, flammability 5`；防御 2 与木盾防御 1 的差距，按硬度算不出来（同为 3）——说明 Tier 1 还缺“厚度/层数”或“覆盖部位”这一维，见 §4 第 3 问 |
| `bread_loaf`（有机物） | `hunger_restore: 25` | `organic: flammability 6`；面包能烧，但烧面包没人会干——有机物的可燃性主要影响的是尸体、草、木材，不是食物本身 |
| `water_flask`（水） | `thirst_restore: 30` | 水不是物品材质，是地块/液体；`TileEffect` 的水已经能灭火，物质表里水只需要 `flammability 0` 这一行 |

验算暴露的问题：**硬度一维不够区分木盾和皮甲**（都是 hardness 3，但防御 1 vs 2）。要么 Tier 1 再加一项（`thickness` 或 `coverage`），要么接受“同硬度不同防具靠手写防御修正”——后者等于物质表没完全取代手写数。见 §4 第 3 问。

---

## 4. 要用户定的事（4 个，逐条定）

1. **物质表做多细？**（survey 开放问题 2）Tier 1 三项（硬度/易燃/密度）先做、Tier 2 温度/相变/导热后加，还是直接上 Tier 2？草案建议 Tier 1。
2. **广告分是数据声明，还是由组件推导？** 方案 A：每个物品/NPC/地块在数据文件里手写 `advertisements: { hunger: 25 }`；方案 B：广告分由已有组件推导（`consumable.hunger_restore` 就是 `hunger` 广告分，不另写一张表）。B 更干净、不重复，但推导规则要写进代码；A 更灵活（同一物品可对不同需求打不同分），但同一数字写两遍。草案建议 B（推导），A 只用于推导不出来的对象（如 NPC 谈话的 `social` 分）。
3. **木盾 vs 皮甲的区分怎么办？** Tier 1 的硬度一维分不开它俩：加 `thickness` / `coverage` 字段，还是接受手写防御修正？或者把 `defense` 理解为“硬度 × 厚度”的推导值，`leather_armor` 只是比 `wooden_shield` 厚？
4. **水要不要做成三重身份？** Qud 里水同时是货币、饮水、外交筹码。Ewil 的 `water_flask` 现在只是消耗品（解渴 30）。水要不要进经济线（当货币）和外交线（水仪式换声望），还是先只当消耗品？这决定 `water_flask` 的广告分是只有 `thirst`，还是同时有 `trade_value` / `reputation`。

---

## 5. 不在这次草案里的

- Top 5 秘密/谣言、Top 6 好感修正栈、Top 7 神器心境、Top 8 Focus 数值桥、Top 9 痕迹半衰期、Top 10 昏迷/目击链：均不在本文档里。注意 Top 9（痕迹半衰期）与 §3.3 的 `Surface.ttl` 有接口关系，但 Top 9 的“事件日志被系统阅读、由旧事生成新机会”不在本次范围。
- survey 开放问题 1（压力系统进基座吗）、3（秘密进 MVP 吗）、4（神器心境保底）、5（经济做到哪层）：不在本文档里；其中问题 5 与 §4 第 4 问（水的三重身份）相关，但经济整体另议。
- 本文档不改 `ecs-architecture.md`、`data/rules/needs.json`、`data/ai/utility_weights.json`、`data/items/*.json`、`data/materials/`（不存在）中的任何一个文件；改动只在用户逐条定下来之后做。

---

## 6. 核对记录（写草案时实测，2026-11-18）

- `data/items/*.json` 10 件全读过：无任何 `material` 字段；`iron_sword` 伤害 6、`iron_spear` 伤害 5、`wooden_shield` 防御 1、`leather_armor` 防御 2，均为手写数。
- `kind: "material"` 仅出现在 `lockpick.json` 与 `machine_part.json`，是物品种类，不是材质。
- `ecs-architecture.md` 全文检索 `Material`：0 处。`Surface`：0 处。`get_available_actions`：§5.0、§5.0.1、§8.2 有定义，枚举规则为硬编码列表，无广告分概念。
- `data/` 与文档全文检索 `bed` / `furniture`：0 处——当前世界没有可打“休息/舒适”广告的家具实体。
- `TileEffect { kind, intensity, ttl }` 与 `Plot` 的 `Soil` / `Moisture` / `Temperature` / `Vegetation` / `MicrobeDensity` 均已在 §2.2–§2.3 确认，写者分别为 `TileEffectSystem` 与 `EnvironmentSystem`。
