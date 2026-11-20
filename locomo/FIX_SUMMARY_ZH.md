# LoCoMo & LoCoMo_refined 错误与纠正汇总

> **语言切换**: [English](./FIX_SUMMARY.md) | **简体中文**  
> **返回主目录**: [`README_ZH.md`](../README_ZH.md) · [`ALL_FIXES_SUMMARY_ZH.md`](../ALL_FIXES_SUMMARY_ZH.md)

---

## 一、目录与文件对照（1:1 镜像结构）

| 原文件路径 (`original/`) | 修正后路径 (`fixed/`) | 文件说明 |
| :--- | :--- | :--- |
| [`original/locomo10.json`](./original/locomo10.json) | [`fixed/locomo10.json`](./fixed/locomo10.json) | 原版 LoCoMo 全部 10 组长对话与 1,986 道问答 |
| [`original/locomo_refined.json`](./original/locomo_refined.json) | [`fixed/locomo_refined.json`](./fixed/locomo_refined.json) | LoCoMo_refined 清洗版全部 10 组对话与 1,382 道问答 |
| [`original/evaluation.py`](./original/evaluation.py) | [`fixed/evaluation.py`](./fixed/evaluation.py) | 原版 LoCoMo 官方词汇指标评测代码 |
| [`original/llm_judge.py`](./original/llm_judge.py) | [`fixed/llm_judge.py`](./fixed/llm_judge.py) | LoCoMo_refined LLM-as-a-Judge 评测代码与提示词 |

---

## 二、逐文件错误与纠正明细

### 1. `original/locomo10.json` → `fixed/locomo10.json`

| 错误类别 | 原文件具体问题 | 纠正方式 | 修复数量 |
| :--- | :--- | :--- | :--- |
| **说话人张冠李戴（Ground-Truth 严重语义错误）** | 单条证据轮次中说话人 A 用第一人称（`I / my`）陈述**自己**的经历或喜好，题目却将其归属给另一位说话人 B：<br>• `Conv 3 QA 119`：`D11:13` Joanna 说自己想写电影，却问 Nate；<br>• `Conv 4 QA 135, 164, 165`：将 John 的学习激励法、会西班牙语、在意大利买烹饪书全问成 Tim；<br>• `Conv 4 QA 166`：`D27:19` Tim 说 *"Harry Potter is my favorite book."*，却问 John 最喜欢的书；<br>• `Conv 6 QA 37`：`D17:28` James 晒自己姐姐与狗的照片，却问 John；<br>• `Conv 7 QA 116, 123, 174, 179, 188`：将 Deborah 的运动感受、与 Anna 看海边日落、因已有安排改期（原答案误写 *"Jolene already had plans"*）、玩猫咪卡牌游戏、提议一起冲浪全部错安给 Jolene；<br>• `Conv 8 QA 144`：`D22:1` Sam 分享徒步照片却问成 Evan；<br>• `Conv 9 QA 88`：`D10:3` Dave 组织公园散步却问成 Calvin。 | 严格对齐证据轮次对话，修正问题与标准答案中的人物指代 | **13 题** |
| **Cat 2 星期错算与日期反引号笔误** | • `Conv 0 QA 5`：Melanie 在 `D2:1` 明确说 *"last Saturday"*，Gold 却写成 `'The sunday before 25 May 2023'`；<br>• `Conv 8 QA 74`：Evan 在 `D20:3` 说 *"last Tuesday"*，Gold 却写成 `'Thursday before December 17, 2023.'`；<br>• `Conv 2 QA 48`：Gold 含反引号笔误 `'The sunday before 3` July 2023'`。 | 分别更正为 `'The Saturday before 25 May 2023'`、`'Tuesday before December 17, 2023.'` 与 `'The Sunday before 31 July 2023'` | **3 题** |
| **单跳/跨轮证据指针偏移、重复与空指针** | • `Conv 0` 6 题（`D7:21`、`D8:5`、`D16:7`、`D17:18`）指向提问轮次而非回答轮次；<br>• `Conv 1 QA 44`（问 Gina 对照片中舞者的评价）误标为 Jon 回复轮 `D1:26`（实为 `D1:25`）；`Conv 2 QA 5` 问 Maria 去海滩时间误标为 John 感叹轮 `D3:15`（实为 `D3:14`）；<br>• `Conv 8 QA 130`、`Conv 9 QA 80`、`Conv 9 QA 111` 漏标跨轮上下文（`D18:4`、`D5:12`、`D19:5`），`Conv 9 QA 5` 重复标注 `'D4:5'`；<br>• **4 道 Cat 3 推理题**（`Conv 0 QA 30, 46`、`Conv 9 QA 39, 42`）`evidence` 为空数组 `[]`。 | 修正偏移、去重冗余指针，并补全全部漏标及 Cat 3 缺失的 `evidence` 指针列表 | **16 题** |
| **Cat 5 拒答题答案字段泄漏** | `Conv 0 QA 167, 178`（Category 5 对抗拒答题）同时存在 `answer='No'` 与 `adversarial_answer='Yes'`，破坏按 `'answer' in qa` 判断题型的评测器。 | 移除 Cat 5 中多余的 `answer` 键 | **2 题** |
| **答案数据类型不统一** | 6 道题的 `answer` 为整型 `int`（如 `2022`、`2`），导致直接调用 `.strip()` / `.lower()` 报错。 | 统一规范化为字符串 `str` | **6 题** |
| **`evidence` 语法损坏与越界悬空指针** | 在 **9 道题**中存在分号拼接 `'D8:6; D9:17'`、冒号错位 `'D:11:26'`、前导零 `'D30:05'`、3 处空格拼接字符串，以及 3 个不存在的悬空指针 `'D'`、`'D10:19'`、`'D4:36'`。 | 拆分并规范化为标准 `D<sess>:<turn>` 数组；移除 3 个悬空指针并记录于 `audit_removed_evidence` | **9 题** |

---

### 2. `original/locomo_refined.json` → `fixed/locomo_refined.json`

| 错误类别 | 原文件具体问题 | 纠正方式 | 修复数量 |
| :--- | :--- | :--- | :--- |
| **`LoCoMo_refined` 漏检残留的说话人张冠李戴** | 原 `LoCoMo_refined` 清洗流程未发现上述说话人颠倒问题，导致其中 **10 道题**原封不动残留在精炼版中（`Conv 4 QA 112, 140, 142`、`Conv 6 QA 25`、`Conv 7 QA 94, 100, 149, 154`、`Conv 8 QA 131`、`Conv 9 QA 73`）。 | 对齐证据对话修正问题与答案的人物指代 | **10 题** |
| **清洗流程反将正确是否题改错（Yes/No 极性反转）** | `Conv 9 QA 37`（*"Based on the conversation, did Calvin and Dave have a meeting in Boston between August and November 2023?"*）原版 `locomo10` 正确标为 `'No'`，但 `LoCoMo_refined` 在 `evidence: []` 缺失的情况下将其反转误改为 `['yes']`（直至 11 月 17 日末轮 `D30:21` 两人仍仅停留在口头约见，从未实际在波士顿见面）。 | 将标准答案恢复为 `['no']`，并补全支撑证据 `['D15:10', 'D17:7', 'D18:11', 'D30:21']` | **1 题** |
| **遗留 `evidence` 语法损坏、轮次偏移与空指针** | 含 **5 题**语法/悬空损坏（`'D8:6; D9:17'`、悬空 `'D'`、`'D30:05'`、2 处空格拼接）与 **10 题**偏移/重复/空指针（`Conv 0` 4 处偏移、`Conv 2 QA 5`、`Conv 9 QA 5` 重复 `'D4:5'`、`Conv 9 QA 65, 95` 漏标，及 **2 道 Cat 3 题** `Conv 0 QA 24`、`Conv 9 QA 37` `evidence: []`）。 | 标准化格式、移除 `'D'`、去重修正并补全全部偏移及空白证据指针 | **15 题**（`10` 偏移/空白 + `5` 语法损坏） |
| **`answer` 列表内元素类型不统一** | 部分 `answer` 候选列表中混入整型 `int`（如 `[2022]`）。 | 列表内元素统一转为 `str` | **6 题** |

---

### 3. `original/evaluation.py` → `fixed/evaluation.py`
* **词袋集合无序 EM 漏洞**：原代码 `set(prediction.split()) == set(ground_truth.split())` 将词序颠倒或重复词判为精确匹配，已修正为规范化字符串相等比较 `normalize_answer(prediction) == normalize_answer(ground_truth)`。
* **停用词误删连词 `"and"`**：原 `remove_articles` 正则为 `r'\b(a|an|the|and)\b'`，已修正为仅移除冠词 `r'\b(a|an|the)\b'`。
* **`rougel_score` 指标张冠李戴**：函数名为 `rougel_score`，返回值却取了 `scores[0]["rouge-1"]["f"]`，已修正为 `scores[0]["rouge-l"]["f"]`。
* **多跳逗号拆分 F1 不惩罚幻觉**：原逻辑对每个 Gold 子项在预测子项中取 `max` 后求平均，模型输出大量错误猜测反而得高分；已修正为对称调用标准 token-level `f1_score`。

### 4. `original/llm_judge.py` → `fixed/llm_judge.py`
* **消除裁判提示词自相矛盾**：原 `refined` 提示词前文规定“包含核心答案且附带额外合理细节判 CORRECT”，后文又规定“包含额外未矛盾项判 WRONG”，且机械拒绝将相对日期换算为等价绝对日期的正确回答；已统一消歧并允许等价绝对/相对日期换算。
