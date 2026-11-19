# Ewil 调研采纳草案：想法→情绪→压力链 + 需求多样化

- 日期：2026-11-18
- 状态：**讨论草案，未采纳，未写进 `ecs-architecture.md`**。只有用户逐条定下来之后，才把采纳的部分写进架构文档。
- 来源：`game-survey.md` 采纳排序 Top 1（想法→情绪→压力链，DF / RimWorld / CK3 / ONI）与 Top 4（需求多样化，人格决定需求权重，DF / Sims 1），对应跨游戏模式 P1、P2、P3、P7。
- 范围：只做这两条，因为它们共用同一根因（48 个性面）且互相咬合；Top 2（对象广告分）、Top 3（物质属性表）、Top 5（秘密/谣言）等不在本文档里，另议。
- 所有数值都是**第一版草案值**，跑模拟时再调。

---

## 1. 现状对照（以 `ecs-architecture.md` v2.0 为准）

| 现在已有的 | 位置 | 缺口 |
|---|---|---|
| `Personality` 48 面，0–10 | §2.5.1、§5.1（49 条行动亲和度） | B 组情绪面（`anxiety`、`anger`、`stress_vulnerability`、`hopefulness`、`emotionality`）已有少量消费者：`panic`（`stress_vulnerability + anxiety`）、`emotional_outburst`（`emotionality + stress_vulnerability`）、`lash_out`、`hold_steady`、`endure`、`rest`。但它们是“系统规则”，不是一条从事件到崩溃的完整链 |
| `Needs` 只有饿 / 渴 / 累 3 项，固定字段 `hunger, thirst, fatigue`，每 tick +1，到 80 强烈倾向解决，到 100 掉血 | §2.4（4）生命与身体、`data/rules/needs.json`（`types: [hunger, thirst, fatigue]`） | 没有社交、独处、创作等需求；所有 NPC 的需求集完全相同 |
| `Memory`：16 条上限，`strength` 每 100 tick −1，低于 10 遗忘；只由事件反应写入 | §2.8 | 事件直接进记忆，中间没有“想法 / 情绪”这一层；没有带持续时间与叠加上限的短期情绪账本（RimWorld 式） |
| 效用 AI 六项权重 `w_need, w_memory, w_reputation, w_goal, w_schedule, w_personality`；`need` 项 = 最急需 / 100 | §5、`data/ai/utility_weights.json` | `need` 项只认 3 项生存需求；新需求进来后归一化方式要跟着改（见 §3.3） |
| 生物数据里的 B 组面目前几乎全是默认值 5 | `data/creatures/*.json`（抽查：铁匠、隐士、旅店老板、农夫、巡逻兵的 `stress_vulnerability`、`anxiety`、`anger` 均为 5） | **如果现在直接加压力系统，所有人的阈值和崩溃倾向都一样**——会重蹈 TES Radiant AI 初版“全员同质”的覆辙（见 `game-survey.md` 第 18 条“跳过”）。加系统之前，必须先按角色分化 B 组面的初始值 |

---

## 2. 草案 A：Thought → Emotion → Stress

### 2.1 新增组件（都在（6）认知与决策类，`Stress` 的身体后果由已有系统结算）

```rust
// 一条事件在某个人心里留下的短期解读。不是记忆本身，是记忆形成之前的中间层。
struct Thought {
    event_id: u64,      // 指向事件日志里的那条事件
    kind: ThoughtKind,  // 如 AteGoodMeal / SawCorpse / WasBetrayed / WasPraised
    tick: i32,          // 产生时间
    valence: i32,       // -100..100，正为好事，负为坏事；由事件 payload × Personality 算出（见 2.2）
    strength: i32,      // 0..100，随时间衰减
}
// 每个实体最多挂 8 条 Thought（RimWorld 式叠加上限）；同 kind 的新想法刷新旧的，不无限叠加。

// 情绪是算出来的状态，不是存下来的性格。ThoughtSystem 是唯一写者。
struct Emotion {
    mood: i32,          // -100..100，由全部 Thought 的 valence × strength 加权求和再归一化
    dominant: EmotionKind, // Joy / Anger / Fear / Sadness / Calm，由 valence 最大的 Thought 决定
    tick: i32,          // 上次重算时间
}

// 压力是累积值。ThoughtSystem 是唯一写者；超过阈值后的行为由 DecisionSystem / 现有系统规则触发。
struct Stress {
    value: i32,         // 0..100
    threshold: i32,     // 见 2.3，由 stress_vulnerability 定
}
```

### 2.2 事件 → 想法：人格改写情绪定价（模式 P3）

同一事件，不同人算出不同 `valence`：

```text
valence = 事件基础分 × 人格修正
- 吃到好饭：基础 +20；`cheerfulness` 高 → ×1.2，`emotionality` 低 → ×0.8
- 目睹尸体：基础 −30；`anxiety` 高 → ×1.5，`bravery` 高 → ×0.6
- 被信任的人背叛：基础 −50；`trust` 高 → ×1.3（越信任伤得越重），`vengefulness` 高 → 同时给复仇类 Thought 加权
- 被帮助：基础 +25；`gratitude` 高 → ×1.3
```

- 规则：每种 `ThoughtKind` 的基础分与人格修正公式写在数据里（`data/ai/thoughts.json`，新建），不在代码里写死。
- 想法产生后，同时记一条 `Memory`（沿用 §2.8 的 16 条上限与衰减）；`Thought` 是短期的（见 2.4），`Memory` 是长期的。两者都指向同一 `event_id`，可互相查证。

### 2.3 压力累积与阈值（草案值）

```text
每 tick：Stress.value += max(0, -mood) / 50        // 心情越差，压力涨得越快；心情为正时不涨
          Stress.value -= 2  （当 mood > 0 且无未处理的负面 Thought 时缓慢回落）
阈值：threshold = 90 - 5 × stress_vulnerability      // stress_vulnerability 5 → 65；9 → 45；1 → 85
```

- `stress_vulnerability` 是 0–10 的面，值越高越容易垮（§2.5.1 原定义不变）。
- 阈值公式是草案值；关键是**阈值因人而异**，不是全世界同一个数。

### 2.4 压力越阈值：按人格选型崩溃，不发明新动作

直接复用 §5.1 已有的系统规则类亲和度，不新增 `ActionKind`：

| 崩溃选型 | 触发条件（草案值） | 复用的已有规则 |
|---|---|---|
| 发火 `lash_out` | `anger ≥ 7` 且 `aggression ≥ 6` | §5.1 `lash_out`（`anger + aggression - patience`） |
| 恐慌逃跑 `panic` | `anxiety ≥ 7` 或 `bravery ≤ 3` | §5.1 `panic`（`stress_vulnerability + anxiety`） |
| 情绪爆发 `emotional_outburst` | `emotionality ≥ 7` | §5.1 `emotional_outburst` |
| 消沉卧床（不做事） | `hopefulness ≤ 3` 且 `stress_vulnerability ≥ 7` | 新增一条系统规则：`AIState` 切到“卧床”，持续 24 tick，期间拒绝工作类 Intent；不是新动作，是 DecisionSystem 的拒绝规则 |
| 忍耐 `endure` | `perseverance ≥ 7` 且 `hopefulness ≥ 6` | §5.1 `endure`：压力照涨，但不触发崩溃，改为 `fatigue` +1/tick（身体代偿） |

- 崩溃后 `Stress.value` 清零到 20（不是 0，留一点余波），并记一条事件；同一实体 100 tick 内不重复触发（冷却，避免一次事件连锁团灭）。
- **阻尼（DF 的改法，见 survey）：** 16 生物的小世界，崩溃不传染：目击者只产生自己的 `Thought`（按 §2.2 定价），不直接复制崩溃者的 `Stress`。级联要靠事件链走，不靠数值传染。

### 2.5 Thought 的衰减（RimWorld 式账本）

```text
Thought.strength 每 10 tick −1；strength ≤ 0 即删除（不记遗忘事件，太琐碎）
同 kind 的 Thought 最多 3 条并存，第 4 条进来时最旧的一条被顶掉
Emotion.mood 每 tick 由现存 Thought 重算，不是累积值
```

- 与 `Memory` 的区别：`Memory` 16 条、每 100 tick −1、低于 10 才遗忘；`Thought` 8 条、衰减快 10 倍、是“现在心情怎么样”，不是“记得发生过什么”。

### 2.6 写者与 Phase（不破坏单写者原则）

| 组件 | 唯一写者 | Phase |
|---|---|---|
| `Thought` | `ThoughtSystem`（新增，事件反应 → 生成 / 衰减） | EventDrain（生成）+ Maintain（衰减） |
| `Emotion` | `ThoughtSystem`（每 tick 由 Thought 重算） | EventDrain 后、Decide 前重算（mood 要进效用 AI 打分） |
| `Stress` | `ThoughtSystem`（按 mood 累积 / 回落） | 同上 |
| `Needs`、`Memories`、`Personality` | 不变（`NeedsSystem`、`MemorySystem`、§5.2 大事件规则） | 不变 |

- 效用 AI 的 `w_need` 项不变；`mood` 以什么权重进打分，见 §3.3——草案建议先不进打分，只影响压力与崩溃，等第一轮模拟确认 48 面本身够不够用之后再决定（对应 survey 开放问题 1 的“后加”选项）。

---

## 3. 草案 B：需求多样化（人格决定需求权重）

### 3.1 原则

- 生存 3 项（饿 / 渴 / 累）对所有人通用，规则不变。
- 新增需求**不是给所有人加同样的清单**，而是按面值“激活”：某面的值达到阈值，该需求才出现在这个实体身上；权重由该面的值决定。避免“所有 NPC 行为同质化”（Sims 的改法，见 survey 第 3 条）。
- 需求定义全部进数据（`data/rules/needs.json` v2），`Needs` 组件从 3 个固定字段改为“需求 id → 0–100”的注册表式结构（与技能开放注册表同构，见 §6）。

### 3.2 新增需求候选表（草案值）

| 需求 id | 激活条件（面值） | 权重公式（草案值） | 衰减 / tick | 缓解方式（进 `get_available_actions` 的候选） |
|---|---|---|---|---|
| `social` 社交 | `sociability ≥ 7` | 权重 = `sociability` / 10 | +1 | 与人交谈 / 一起吃饭（`Talk`） |
| `solitude` 独处 | `independence ≥ 7` | 权重 = `independence` / 10 | +1（周围有人时 +2） | 独自待着 / 去没人的地方（`Rest` / `Move`） |
| `craft` 做东西 | `diligence ≥ 7` 或 `artistic ≥ 7` | 权重 = max(`diligence`, `artistic`) / 10 | +1 | 做活 / 精修（`Craft`，复用 §5.1 `work_craft` / `refine`） |
| `thrill` 刺激 | `excitement_seeking ≥ 7` | 权重 = `excitement_seeking` / 10 | +1 | 探险 / 找新鲜（`explore_ruin` / `seek_novelty`） |
| `order` 整洁 | `orderliness ≥ 7` | 权重 = `orderliness` / 10 | +1（周围脏乱时 +2） | 收拾 / 整理（`Craft` 清理类） |
| `romance` 亲密 | `love_propensity ≥ 7` | 权重 = `love_propensity` / 10 | +1 | 求偶 / 倾诉（`romance` / `confide`） |

- 到 80 同样触发“强烈倾向解决”，到 100 不掉血（与生存需求不同）：改为 `Stress.value +5 / tick`（心理需求长期不满足 → 涨压力，走草案 A 的链，而不是直接扣血）。
- 每个生物最多同时激活 2 项新增需求（取权重最高的两项），避免需求清单膨胀。

### 3.3 对效用 AI 的影响

- `w_need` 项的归一化改为：`最急需（按权重加权后的缺口）/ 100`，即 `need = max_over_needs(缺口 × 权重) / 100`。生存需求权重恒为 1，新增需求权重按 §3.2 的公式来。
- `get_available_actions` 的候选列表要包含缓解新增需求的行动（上表最后一列）；没有可缓解的对象在附近时，该需求不进候选（与 §5.1 “想偷但附近没东西就不进候选”同一规则）。

### 3.4 用现有生物数据验算（取自 `data/creatures/*.json` 的真实面值）

| 生物 | 面值 | 按 §3.2 会激活什么 |
|---|---|---|
| 旅店老板阿桂 | `sociability 9, friendliness 8, humor 8, cheerfulness 7` | `social`（权重 0.9）——整天跟人打交道，符合人设；第二项无（其余面均 < 7） |
| 机械教团隐士锈眼 | `independence 9, caution 8, sociability 2` | `solitude`（权重 0.9）——独处是刚需；`social` 不激活（2 < 7），人越多他越难受 |
| 铁匠 | `diligence 7, sociability 6` | `craft`（权重 0.7）——不打铁就难受；`social` 不激活（6 < 7，差一点） |
| 农夫老田 | `diligence 8` | `craft`（权重 0.8，种田 / 做活即缓解） |
| 巡逻兵铁柱 | `bravery 7, aggression 6` | 无新增需求（`thrill` 要 `excitement_seeking ≥ 7`，他没有）——说明阈值 7 会漏掉“好斗但不好刺激”的人，阈值与面值的对应关系还要调 |

- 验算暴露的问题：`thrill`、`order`、`romance` 在现有 16 个生物里几乎无人激活——因为除个别面外，大部分面的初始值都是默认 5。**需求多样化之前，要先把各角色的面值按人设分化**（与 §1 最后一段同一个前提）。

### 3.5 一个缺口：祈祷 / 信仰需求没有对应的面

- DF 的需求清单里有“祈祷”，但 Ewil 的 48 面里没有信仰 / 虔诚面（§2.5.1 的 8 组里没有这一组）。
- 草案建议：**这次不加祈祷需求**。要加就得先加一个面，而“没有消费者的面不许存在”（§2.5.1 的规则）——新面必须同时配上行动亲和度与系统规则，是一次单独的讨论，不塞进本次草案。

---

## 4. 要用户定的事（3 个，逐条定）

1. **进基座，还是等一轮模拟之后？**（对应 survey 开放问题 1）草案 A 的 3 个新组件 + 1 条规则，是现在就进基座，还是等第一轮模拟跑起来、确认 48 面人格本身够不够用之后再加？草案本身不预设答案；如果选“后加”，§2.6 的 `ThoughtSystem` 与 §3 的 `needs.json` v2 都先不做。
2. **崩溃的代价接受到什么程度？** 消沉卧床 24 tick、发火可能伤人，都是真实后果；16 生物小世界里一次崩溃会不会太重？要不要给崩溃加保底（例如卧床期间 `Stress` 回落更快，或发火只砸东西不伤人）？
3. **B 组面的分化先做吗？** 无论选“现在进”还是“后加”，§1 与 §3.4 指出的前提都成立：`stress_vulnerability`、`anxiety`、`anger` 等面现在全员默认 5，不分化就加系统等于没加。是否先做一轮“按角色分化 B 组面值”的数据调整（只改 `data/creatures/*.json`，不动架构）？

---

## 5. 不在这次草案里的

- Top 2 对象广告分（`get_available_actions` 的实现方式）：与草案 B 的 §3.3 有接口关系，但实现本身另议。
- Top 3 物质属性表、Top 5 秘密/谣言、Top 7 神器心境、Top 8 Focus 数值桥、Top 9 痕迹半衰期、Top 10 昏迷/目击链：均不在本文档里。
- survey 开放问题 2–5（物质表多细、秘密进不进 MVP、神器心境保底、经济做到哪层）：不在本文档里。
- 本文档不改 `ecs-architecture.md`、`data/rules/needs.json`、`data/ai/utility_weights.json`、`data/creatures/*.json` 中的任何一个文件；改动只在用户逐条定下来之后做。
