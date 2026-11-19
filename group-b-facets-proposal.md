# Ewil B 组情绪面分化方案（提案，未应用）

- 日期：2026-11-18
- 状态：**提案，未应用。`data/creatures/*.json` 一个字没改，`ecs-architecture.md` 没动。** 用户点头（或逐条改）之后再应用。
- 针对：`survey-adoption-draft.md` §4 第 3 问——“B 组面的分化先做吗？”这是那一步的具体数值方案，不是新讨论。
- 范围：只改 B 组 6 个面（`cheerfulness, anxiety, anger, stress_vulnerability, hopefulness, emotionality`），其余 42 个面不动。

## 1. 核对过的现状（2026-11-18，从 `data/creatures/*.json` 实测）

| 面 | 现状 |
|---|---|
| `stress_vulnerability` | 16 个生物**全是 5**。按草案公式 `threshold = 90 − 5 × sv`，全员阈值都是 65 |
| `anger` | 16 个生物**全是 5** |
| `anxiety` | 15 个是 5，只有小孩（`villager_child`）是 3 |
| `cheerfulness` | 已有分化：小孩 8、厨子 7、旅店老板 7，其余 5 |
| `hopefulness` | 已有分化：小孩 8、巡逻兵 1 号 6、农夫 6，其余 5 |
| `emotionality` | 已有分化：医生 6，其余 5 |

所以问题比“全员默认 5”更精确一点：`stress_vulnerability` 和 `anger` 是全员同质，`anxiety` 几乎同质；另外三个面已有零星分化，本方案予以保留，不推倒重来。

## 2. 提案表

数值依据：每个生物已有的非默认面 + `desc` 人设（见第 3 节逐条理由）。取值范围 3–8，没用 9/10 的极端值；已分化的面（小孩的 `cheerfulness 8` 等）原样保留。

`threshold = 90 − 5 × stress_vulnerability`（草案 A §2.3）。崩溃选型按草案 A §2.4 的条件验算。

| 生物 | cheerfulness | anxiety | anger | stress_vuln. | hopefulness | emotionality | 阈值 | 验算出的崩溃选型 |
|---|---|---|---|---|---|---|---|---|
| 隐士锈眼 `mechanist_hermit` | 4 | 7 | 3 | 7 | 5 | 4 | 55 | panic（anxiety 7） |
| 巡逻兵铁柱 `patrol_guard_1` | 6 | 4 | 6 | 4 | 6 | 4 | 70 | — |
| 巡逻兵石头 `patrol_guard_2` | 5 | 3 | 7 | 5 | 5 | 5 | 65 | lash_out（anger 7 + aggression 6） |
| 拾荒头目瘦猴 `scavenger_1` | 6 | 5 | 6 | 6 | 6 | 5 | 60 | — |
| 拾荒者爪子 `scavenger_2` | 5 | 3 | 8 | 4 | 5 | 6 | 70 | lash_out（anger 8 + aggression 8） |
| 拾荒者酸水 `scavenger_3` | 4 | 6 | 6 | 7 | 4 | 6 | 55 | —（见 §4 缺口 1） |
| 学徒小锤 `villager_apprentice` | 7 | 4 | 4 | 5 | 7 | 6 | 65 | — |
| 铁匠 `villager_blacksmith` | 5 | 4 | 5 | 4 | 6 | 4 | 70 | — |
| 小孩小豆 `villager_child` | 8 | 3 | 3 | 4 | 8 | 7 | 70 | panic（bravery 3）+ emotional_outburst（emotionality 7） |
| 厨子阿香 `villager_cook` | 7 | 4 | 3 | 4 | 6 | 6 | 70 | — |
| 医生白芷 `villager_doctor` | 5 | 6 | 2 | 6 | 6 | 6 | 60 | — |
| 村长石爷 `villager_elder` | 6 | 3 | 3 | 3 | 6 | 5 | 75 | — |
| 农夫老田 `villager_farmer` | 5 | 4 | 3 | 4 | 6 | 4 | 70 | endure（perseverance 7 + hopefulness 6） |
| 猎手夜鹰 `villager_hunter` | 5 | 3 | 4 | 3 | 6 | 4 | 75 | — |
| 旅店老板阿桂 `villager_innkeeper` | 7 | 4 | 4 | 4 | 6 | 7 | 70 | emotional_outburst（emotionality 7） |
| 商人阿珍 `villager_merchant` | 6 | 5 | 5 | 6 | 6 | 6 | 60 | — |

合计改动 70 / 96 个面值（其余 26 个保持原值，包括所有已分化的面）。

## 3. 逐条理由（一人一行）

- **隐士锈眼**：`bravery 3, caution 8, independence 9`——胆小谨慎又独居，焦虑拉满（7）、抗压最差一档（7，阈值 55），发火低（3，不惹事，躲起来）。
- **铁柱**：`bravery 7, loyalty 8, dutifulness 7`——受过训的巡逻兵，稳（sv 4，阈值 70），`anger 6` 离发火线（7）差一步，是“守规矩的硬汉”。
- **石头**：`bravery 8, aggression 6, cruelty 6, caution 4`——16 人里最莽的之一，`anger 7` 是唯一靠 `anger + aggression` 触发 `lash_out` 的村民（另一个是爪子）。
- **瘦猴**：`greed 8, envy 7, trust 3`——投机头目，`sv 6`（阈值 60，算计多、怕翻车），`anger 6` 没到发火线，先算账再动手。
- **爪子**：`aggression 8, cruelty 7, caution 3`——`anger 8` 全场最高，`lash_out` 的另一人选；但 `sv 4`，他是施压者不是承压者。
- **酸水**：`aggression 6, cruelty 6, patience 4`——`sv 7`（阈值 55，和隐士并列最低），`anxiety 6`、`hopefulness 4`，神经质又没指望，是全场最容易垮的人（但垮成什么样见 §4）。
- **小锤**：`diligence 8, curiosity 8, excitement_seeking 7`——年轻学徒，`cheerfulness 7, hopefulness 7`，`sv 5` 中间值。
- **铁匠**：`diligence 7, perfectionism 7, patience 6`——手艺人 steady，`sv 4`（阈值 70），`emotionality 4`，闷头干活型。
- **小豆**：已有的 `cheerfulness 8, hopefulness 8, anxiety 3` 保留；`sv 4`（孩子恢复快）、`emotionality 7`（情绪外露，触发 `emotional_outburst` 候选）、`anger 3`。
- **阿香**：`sociability 8, cheerfulness 7` 保留；`sv 4`、`anger 3`，厨房是全村情绪最稳的地方之一。
- **白芷**：`altruism 8, caution 7, patience 7`——医生扛别人的苦，`anxiety 6, sv 6`（阈值 60， burnout 风险），`anger 2` 全场最低。
- **石爷**：`loyalty 8, patience 7, thoughtfulness 8`——村长定海神针，`sv 3`（阈值 75，全场最抗压）、`anxiety 3, anger 3`。
- **老田**：`diligence 8, perseverance 7, patience 7`——`sv 4`，且是唯一触发 `endure` 的人（`perseverance 7 + hopefulness 6`）：压力照涨，但不崩溃，改由身体代偿。
- **夜鹰**：`bravery 7, independence 7, patience 7`——独行猎手，`sv 3`（阈值 75，和村长并列最抗压）、`anxiety 3`。
- **阿桂**：已有的 `cheerfulness 7, sociability 9, humor 8` 保留；`emotionality 7`（情绪外露，触发 `emotional_outburst` 候选），`sv 4`。
- **阿珍**：`greed 8, ambition 7`——钱的事最上心，`sv 6`（阈值 60，行情一波动就紧张），其余面温和。

## 4. 验算暴露的两个缺口（草案 A 本身的问题，不是数值问题）

1. **10 / 16 的人越阈值后没有定义好的崩溃选型。** 按 §2.4 的条件，只有 6 个人能对上号：`lash_out` 2 人（石头、爪子）、`panic` 2 人（隐士、小豆）、`emotional_outburst` 2 人（小豆、阿桂）、`endure` 1 人（老田），`bedridden` 0 人（小豆一人占两个选型）。剩下 10 人——铁柱、瘦猴、酸水、小锤、铁匠、阿香、白芷、石爷、夜鹰、阿珍——越阈值后草案没写会发生什么。建议：草案 A 需要一个默认 fallback（例如“退回休息 / 拒绝工作类 Intent”，或把 `endure` 放宽为默认），否则阈值对多数人只是个数字。
2. **小孩会 panic，不是因为焦虑，而是因为胆小。** `panic` 的条件是 `anxiety ≥ 7 或 bravery ≤ 3`，小豆 `anxiety 3` 但 `bravery 3`，按字面规则他一受惊就跑。这符合“孩子受惊就跑”的人设，但规则本身是把“胆小”等同于“易恐慌”——如果用户不想这样，`panic` 条件应改成只看 `anxiety`，或 `bravery` 只做加权不做触发。

## 5. 分布总览（验算后）

- 阈值分布：55（2 人：隐士、酸水）、60（3 人：瘦猴、白芷、阿珍）、65（2 人：石头、小锤）、70（7 人）、75（2 人：石爷、夜鹰）。从全员 65 变成 55–75 的梯度，跨度 20。
- `stress_vulnerability` 分布：3（2 人）、4（7 人）、5（2 人）、6（3 人）、7（2 人）。
- `anger` 分布：2（1 人）、3（5 人）、4（3 人）、5（2 人）、6（3 人）、7（1 人）、8（1 人）。
- `anxiety` 分布：3（5 人）、4（6 人）、5（2 人）、6（2 人）、7（1 人）。
- 没有人同时满足 `bedridden`（`hopefulness ≤ 3 且 sv ≥ 7`），所以默认没有人会卧床不起——如果用户想要至少一个“垮到卧床”的例子，酸水是候选（把 `hopefulness` 从 4 改到 3 即可，但那是改人设，需用户定）。

## 6. 怎么应用（用户点头后）

- 只改 `data/creatures/*.json` 里的 B 组 6 个面，按 §2 表逐文件改值；不动架构文档、不动 `needs.json`、不新建组件。
- 应用后复核：16 个文件 JSON 能解析、48 面齐全、阈值按公式重算一遍与 §2 表一致。
- 本方案不预设草案 A/B 是否进基座（草案 §4 第 1 问）；即使选“等一轮模拟之后再加”，这份分化也可以先做，因为它是数据准备，不是系统接入。
