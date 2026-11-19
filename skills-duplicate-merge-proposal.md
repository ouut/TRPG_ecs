# 技能注册表近重复合并提案（未应用）

- 日期：2026-11-18
- 状态：**提案，未应用。`data/` 一个字没改，`ecs-architecture.md` 没动。** 用户选定方案 A 或 B 之后再应用。
- 针对：MEMORY 里记的剩余数据缺口——技能注册表里 `sneaking` / `stealth` 与 `healing` / `medicine` 两对近重复还没合并。

## 1. 核对过的现状（2026-11-18，从 `data/` 实测）

| 技能 id | 文件 | 被谁用 | tags |
|---|---|---|---|
| `sneaking` | `data/skills/sneaking.json` | 3 个拾荒者（`scavenger_1/2/3.json`） | quiet, shadow, move |
| `stealth` | `data/skills/stealth.json` | 无生物用；只在 `data/rules/skills.json` 12 个种子示例里 | quiet, shadow, hide |
| `healing` | `data/skills/healing.json` | 医生（`villager_doctor.json`） | wound, herb, medicine |
| `medicine` | `data/skills/medicine.json` | 无生物用；只在 `data/rules/skills.json` 12 个种子示例里 | wound, herb, cure |

- 两对的 `governs` 与 XP 曲线完全相同：`sneaking`/`stealth` 都是 `dex + perception`，`xp_on_success 10 / xp_on_fail 3`；`healing`/`medicine` 都是 `int + dex（手）`，同一条 XP 曲线。名字也相同：`sneaking` 与 `stealth` 的 `name` 都是“潜行”。
- `data/rules/leveling.json` 提到 `stealth` 一次，只是举例句子（“never levels stealth”），不是引用。
- 注册表其余技能 id 的命名习惯是动名词：`smithing`、`guarding`、`farming`、`hunting`、`cooking`、`brewing`、`tinkering`、`tracking`、`lockpicking`、`crafting`、`trading`、`survival` 除外多为 -ing 形。`stealth` / `medicine` 是名词，与这个习惯不一致。

## 2. 方案 A（推荐）：保留生物在用的 id，删种子里的重复项

- 删 `data/skills/stealth.json` 与 `data/skills/medicine.json`。
- `data/rules/skills.json` 的 12 个种子示例里删 `stealth` 与 `medicine` 两条（种子剩 10 条；注意种子列表只是示例、不是上限，见该文件 `note`）。
- `sneaking.json` 的 tags 补上 `hide`（并为 quiet, shadow, move, hide）；`healing.json` 的 tags 补上 `cure`（并为 wound, herb, medicine, cure）。
- 生物文件**不用动**：3 个拾荒者继续用 `sneaking`，医生继续用 `healing`。
- 依据：改动面最小（2 个文件删除 + 1 个文件删 2 行 + 2 个文件各加 1 个 tag）；且与注册表 -ing 命名习惯一致。

## 3. 方案 B：改用通用 TRPG 的标准名，生物跟着改

- 删 `data/skills/sneaking.json` 与 `data/skills/healing.json`；保留 `stealth.json` 与 `medicine.json`（tags 同样合并）。
- 改 4 个生物文件：`scavenger_1/2/3.json` 的技能 id `sneaking` → `stealth`；`villager_doctor.json` 的 `healing` → `medicine`。
- `data/rules/skills.json` 不用动（种子列表里本来就是 `stealth` / `medicine`）。
- 依据：`stealth` / `medicine` 是通用 TRPG 里更标准的术语，新人一眼能懂；代价是改 4 个生物文件，且 `stealth` / `medicine` 与注册表 -ing 习惯不一致。

## 4. 应用后复核（任选其一方案后）

- 23 个技能文件 → 21 个，`data/skills/*.json` 全部能解析，id 无重复。
- `data/rules/skills.json` 列出的每个 id 在 `data/skills/` 里都有对应文件；每个生物文件引用的技能 id 都有对应文件。
- `grep -rn "stealth\|sneaking\|healing\|medicine" data/ ecs-architecture.md` 只剩一个 id 每对，无残留引用。
