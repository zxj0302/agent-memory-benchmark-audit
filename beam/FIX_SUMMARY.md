# BEAM 错误与纠正汇总

## 一、目录与文件对照（1:1 镜像结构）

| 原文件路径 (`original/`) | 修正后路径 (`fixed/`) | 文件说明 |
| :--- | :--- | :--- |
| [`original/probing_questions/`](./original/probing_questions/) | [`fixed/probing_questions/`](./fixed/probing_questions/) | `100K`(20) + `500K`(35) + `1M`(35) + `10M`(10) 全量 **100 组对话**共 2,000 道探测题与 Rubric（其中 30 组含缺陷文件已修正，70 组无缺陷文件原样保留，可直接整目录替换） |
| [`original/prompts.py`](./original/prompts.py) | [`fixed/prompts.py`](./fixed/prompts.py) | LLM-Judge 裁判提示词与生成提示词模板 |
| [`original/compute_metrics.py`](./original/compute_metrics.py) | [`fixed/compute_metrics.py`](./fixed/compute_metrics.py) | 10 大记忆评测维度的指标计算与裁判调用逻辑 |
| [`original/run_evaluation.py`](./original/run_evaluation.py) | [`fixed/run_evaluation.py`](./fixed/run_evaluation.py) | 批量评测入口脚本 |
| [`original/long_term_memory_methods.py`](./original/long_term_memory_methods.py) | [`fixed/long_term_memory_methods.py`](./fixed/long_term_memory_methods.py) | 长程记忆基线方法推理代码 |

> **说明**：BEAM 原始背景长对话 `chats/*/*/chat.json`（总计约 4.1 GB）在审计中作为只读事实基准使用，未做改动；完整背景对话可从 [BEAM 官方仓库/数据集](https://github.com/mohammadtavakoli78/BEAM) 获取后与本目录 `fixed/probing_questions/` 直接配合使用。

---

## 二、逐文件错误与纠正明细

### 1. `original/probing_questions/` → `fixed/probing_questions/`（涉及 30 个会话文件、34 道题）

由于官方 `run_evaluation.py` 评分时**仅向裁判模型传入 `rubric`，完全忽略 `answer` 与 `ideal_answer`**，任何 `rubric` 污染或算术错误都会导致答对的模型被直接判错：

| 错误类别 | 具体问题定位 | 纠正方式 | 修复数量 |
| :--- | :--- | :--- | :--- |
| **跨会话 Rubric 粘贴污染（致命）** | `500K/8 CR[0]`（椭圆与双曲线数学题）和 `500K/9 CR[0]`（等比数列数学题）的 `rubric` 原样粘贴了 `1M/2 CR[1]` 的软件测试评分标准：`"LLM response should mention: you said you created Cypress tests for user login and personalized feed with a passing rate of 92%"`。 | 依据各自真实对话证据与 `ideal_answer` 重写对应的数学矛盾 `rubric` | **2 题** |
| **凭空捏造幻觉题 & 日期丢失锚点幻觉题** | • `500K/2 TR[0]`：对话主题为 Python 简历解析器（全文 `hotel` 出现 **0 次**），却问“4月10日订酒店与两天后入住相隔几天”；<br>• `1M/7 TR[0]`：对话主题为常微分方程（ODE），却问不存在的“项目启动会与终审会相隔 22 天”；<br>• `500K/31 TR[1]`：`chat.json` 生成时漏掉了大纲中的日期（2月20日与3月2日，相差11天），出题模型竟把消息末尾的大纲编号 `->-> 1,18` 当成 `18 days`；<br>• `1M/6 TR[1]`：标准答案混入 `"inferred from batch anchors"` 内部元提示词。 | 将 3 道不可答幻觉题替换为各自 `chat.json` 中真实带明确日期的事件问答（保留 `replaced_defective_question` 与 `defect_reason` 供溯源），并清理 `1M/6 TR[1]` 元泄漏文本 | **4 题** |
| **`answer` / `calculation` 与 `rubric` 自相矛盾** | • `10M/5 MSR[0]`：`reasoning_steps` 写 `60+50+50=160 minutes`，核查原对话（`id=74` 1小时 + `id=3276` 50分钟，`id=3277` 实为助手回复）真实总时长为 **110 分钟**，而 `answer` 与 `rubric` 却要求 `350 minutes`；<br>• `10M/3 TR[1]`（算式/答案 `11 days` vs Rubric `12 days`）；<br>• `500K/19 TR[0]`（算式 `23 days` vs 答案/Rubric `9 days`）；<br>• `500K/9 TR[1]`（答案 `130 hours` vs 算式/Rubric `170 hours`）；<br>• `500K/16 TR[1]`（答案 `21 days` vs Rubric `11 days`）；<br>• `500K/22 TR[0]`（答案 `30 months` vs Rubric `33 months`）；<br>• `100K/1 TR[0]`（答案 `4 weeks` vs Rubric `8 weeks`）；<br>• `100K/14 TR[0]`（答案 `11 days` vs Rubric `17 days`）；<br>• `1M/22 TR[0]`（答案 `13 days` vs Rubric `12 days`）；<br>• `1M/24 TR[0]`（Rubric 将 `April 27` 误写为 `April 17`）；<br>• `100K/12 TR[1]`（原文明确说截至5月31日写了 `58 days`，Rubric 却错推起点为 `April 5`）。 | 逐题回查原对话轮次与公历算式，统一对齐 `answer`、`calculation_required`、`reasoning_steps` 与 `rubric` | **11 题** |
| **日历跳月与月份天数幻觉** | • `100K/17 TR[1]`：将 `April 20` 至 `July 5`（实为 **76 天**）漏算整整一个月写成 `46 days`；<br>• `500K/34 TR[0]`：把 3 月当成 28 天，将 `March 20` 至 `April 14`（25 天 = **3 周 4 天**）写成 `3 weeks and 1 day`；<br>• `500K/12 TR[1]`：将 `March 22` 至 `April 12`（21 天）误算为 `20 days`。 | 按真实公历天数修正答案、算式与 `rubric` | **3 题** |
| **2024 闰年 2 月 29 日系统性漏算** | 全部跨越 2024 年 2 月的天数差题目（`1M/12 TR[0]` `123→124`、`1M/18 TR[1]` `46→47`、`10M/5 TR[1]` `27→28`、`500K/23 TR[0]` `48→49`、`1M/34 TR[0]` `136→137`）均按平年 28 天计算，系统性少算 1 天。 | 计入 2024 年 2 月 29 日，统一修正答案、算式与 `rubric` | **5 题** |
| **`event_ordering` 元数据与损坏引用** | 9 处 `total_mentions != len(rubric)`（`100K/14`、`500K/15,22,29`、`1M/1,7,30,32`、`10M/1`）及 `10M/6` `event_ordering[0]` 中的损坏引用字符串 `'--'`。 | 对齐 `total_mentions` 计数并移除非法引用 | **10 题**（其中 `100K/14`、`500K/22`、`1M/7` 与上表共享文件） |

---

### 2. `prompts.py` & `compute_metrics.py` → `fixed/prompts.py`, `fixed/compute_metrics.py`
* **裁判提示词漏传问题文本 `<question>`（致命评测漏洞）**：原 `unified_llm_judge_base_prompt` 要求裁判判断模型回复是否准确回应了 `the QUESTION`，但模板中只有 `- RESPONSE` 与 `- RUBRIC` 占位符，从未注入 `probing_question`；已在 `fixed/prompts.py` 中补齐 `- QUESTION: <question>`，并在 `fixed/compute_metrics.py` 全部 10 个 `evaluate_*` 函数中完成替换注入。
* **`event_ordering` 切分脆弱性与零除保护**：兼容单行编号列表（`1) ... 2) ...`）与多行列表切分，并对空 `rubric` 增加零除保护。

### 3. `run_evaluation.py` & `long_term_memory_methods.py`
* **`run_evaluation.py`**：将 `initialize_models()` 移出每道题的内层循环，避免评测时重复初始化加载 3 个 SentenceTransformer 模型达 2,000 次。
* **`long_term_memory_methods.py`**：原代码 `messages = saved_messages` 使用列表引用别名而非拷贝，导致 `.append()` 原地污染同一会话后续第 2～20 题的历史上下文；已修正为 `messages = list(saved_messages)`。
