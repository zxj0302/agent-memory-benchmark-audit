# BEAM 错误与纠正汇总

> **语言切换**: [English](./FIX_SUMMARY.md) | **简体中文**  
> **返回主目录**: [`README_ZH.md`](../README_ZH.md) · [`ALL_FIXES_SUMMARY_ZH.md`](../ALL_FIXES_SUMMARY_ZH.md)

---

## 一、目录与文件对照（1:1 镜像结构）

| 原文件路径 (`original/`) | 修正后路径 (`fixed/`) | 文件说明 |
| :--- | :--- | :--- |
| [`original/probing_questions/`](./original/probing_questions/) | [`fixed/probing_questions/`](./fixed/probing_questions/) | `100K`(20) + `500K`(35) + `1M`(35) + `10M`(10) 全量 **100 组对话**共 2,000 道探测题与 Rubric（其中 **67 组含缺陷会话文件、共 115 道题**已修正，33 组无缺陷文件原样保留，可直接整目录替换） |
| [`original/prompts.py`](./original/prompts.py) | [`fixed/prompts.py`](./fixed/prompts.py) | LLM-Judge 裁判提示词与生成提示词模板 |
| [`original/compute_metrics.py`](./original/compute_metrics.py) | [`fixed/compute_metrics.py`](./fixed/compute_metrics.py) | 10 大记忆评测维度的指标计算与裁判调用逻辑 |
| [`original/run_evaluation.py`](./original/run_evaluation.py) | [`fixed/run_evaluation.py`](./fixed/run_evaluation.py) | 批量评测入口脚本 |
| [`original/long_term_memory_methods.py`](./original/long_term_memory_methods.py) | [`fixed/long_term_memory_methods.py`](./fixed/long_term_memory_methods.py) | 长程记忆基线方法推理代码 |

> [!NOTE]
> BEAM 原始背景长对话 `chats/*/*/chat.json`（总计约 4.1 GB）在审计中作为只读事实基准使用，未做改动；完整背景对话可从 [BEAM 官方仓库](https://github.com/mohammadtavakoli78/BEAM) 获取后与本目录 `fixed/probing_questions/` 直接配合使用。

---

## 二、逐文件错误与纠正明细

### 1. `original/probing_questions/` → `fixed/probing_questions/`（涉及 67 个会话文件、115 道题）

由于官方 `run_evaluation.py` 评分时**仅向裁判模型传入 `rubric`（忽略 `answer` 与 `ideal_answer`）**，且检索召回率（Recall@K）与证据归因实验直接依赖 `source_chat_ids` Golden Evidence，上述两类损坏均会直接造成评测失真：

| 错误类别 | 具体问题定位 | 纠正方式 | 修复数量 |
| :--- | :--- | :--- | :--- |
| **`chat.json` 合成漏注入矛盾与知识更新（致命不可答题）** | 共 **16 道题**（**7 道 `contradiction_resolution`** + **9 道 `knowledge_update`**）在大纲 `plan_new.txt` 中写了矛盾或更新条目，但合成长对话 `chat.json` 时**完全漏生成该轮或被扭曲为元文本/反问句**，导致即便通读全文也根本不存在第二方陈述或更新值：<br>• **`CR` 漏注入（7 题）**：`1M/2 CR[0]`（HTTP 重试逻辑）、`1M/2 CR[1]`（Cypress 测试）、`1M/4 CR[0]`（里程碑规划）、`1M/33 CR[1]`（Riverside Pavilion 踩点）、`1M/34 CR[1]`（`1086` 轮把否定扭曲为 `"without contradicting our previous agreement"`）、`10M/3 CR[1]`（加密行为日志）、`10M/5 CR[1]`（与 Devin 讨论自伴算子扩张）；<br>• **`KU` 漏注入（9 题 + 1 题部分漏更）**：`100K/9 KU[1]`（`May 22` 从未出现，全文仅 `May 20`）、`100K/11 KU[0]`（`82` 轮用户反问助手改期到了哪天，全文无 `March 27`）、`100K/12 KU[0]`（`April 22` 缺失）、`100K/14 KU[1]`（`30 cupcakes` 缺失，仅 `24`）、`500K/7 KU[1]`（数论归纳 `98%` 缺失）、`500K/34 KU[1]`（`14 players` 缺失，仅 `12`）、`1M/6 KU[1]`（Jacobian `17 minutes` 缺失）、`1M/11 KU[0]`（`30` 个领英连接缺失）、`1M/24 KU[1]`（`April 4` 烹饪课缺失），以及 `10M/1 KU[1]`（负载均衡实为 `12 tasks` + 更新目标 `85%`）。 | 将 16 道单边缺失不可答题替换为同会话 `chat.json` 中 100% 真实双边存在的矛盾/更新问答（保留 `replaced_defective_question` 与 `defect_reason` 供溯源），并将 `10M/1 KU[1]` 对齐至原文 | **17 题** |
| **Golden Evidence (`source_chat_ids`) 损坏、双边重复、悬空或错位（跨 5 大类）** | 根源位于上游生成源码 `BEAM/src/beam/main.py`（`L2180–2191` & `L2294–2304`）：当 `batch_numbers` 与 `bullet_numbers` 逗号切分长度不等（如 `"1, 2"` 对 `"5, 6, 34"`）时，`str(i+1) == batch_number` 匹配失败导致传给生题模型的轮次为空或仅剩单边；同时插入矛盾/更新条目后大纲行号顺延，造成精确编号匹配偏移到相邻无关轮次：<br>• **`contradiction_resolution`（27 题）**：正反两侧指向完全相同/子集 ID（如 `[66]` vs `[66]`、`[730]` vs `[730]`）、单侧为空 `[]`（`1M/1 CR[1]`）、两侧全指向正向轮次漏掉 `"never"` 轮次，或偏移到完全无关轮次（并顺带修复 `1M/17 CR[1]` 题干笔误 `"Have you I used"` → `"Have I used"`）；<br>• **`knowledge_update`（23 题）**：`1` 处 `None`（`500K/1 KU[0]`）、`7` 处 `updated_info: []` 为空、`7` 处 `original_info == updated_info` 重复、`8` 处偏移到无关轮次；<br>• **`summarization`（12 题）** 与 **`preference_following`（2 题）**：`source_chat_ids` 为 `[]` 或 `None`（并修复 `100K/18 SUM[1]` 第三人称代词泄漏 `"as she prepared"` / `"her portfolio"`）；<br>• **`event_ordering`（1 题）**：`10M/6 EO[0]` 含占位符 `['--']`，且因 `"batch 10" < "batch 7"` 字典序排序错误把第 10 批（`3838`）排在第 7、8 批之前、把 `plan-1`（`918/1004`）漏标在末尾。 | 逐题回查 `chat.json` 原始用户消息，补齐并校正全部 65 道题的精确 `source_chat_ids`（全集 1,800 道非拒答题实现 0 空指针、0 非整型、0 悬空 ID） | **65 题** |
| **跨会话 Rubric 粘贴污染（致命）** | `500K/8 CR[0]`（椭圆与双曲线数学题）和 `500K/9 CR[0]`（等比数列数学题）的 `rubric` 原样粘贴了 `1M/2 CR[1]` 的软件测试评分标准：`"LLM response should mention: you said you created Cypress tests for user login and personalized feed with a passing rate of 92%"`。 | 依据各自真实对话证据与 `ideal_answer` 重写对应的数学矛盾 `rubric` | **2 题** |
| **凭空捏造幻觉题 & 日期丢失锚点幻觉题** | • `500K/2 TR[0]`：对话主题为 Python 简历解析器（全文 `hotel` 出现 **0 次**），却问“4月10日订酒店与两天后入住相隔几天”；<br>• `1M/7 TR[0]`：对话主题为常微分方程（ODE），却问不存在的“项目启动会与终审会相隔 22 天”；<br>• `500K/31 TR[1]`：`chat.json` 生成时漏掉了大纲中的日期（2月20日与3月2日，相差11天），出题模型竟把消息末尾的大纲编号 `->-> 1,18` 当成 `18 days`；<br>• `1M/6 TR[1]`：标准答案混入 `"inferred from batch anchors"` 内部元提示词。 | 将 3 道不可答幻觉题替换为各自 `chat.json` 中真实带明确日期的事件问答（保留 `replaced_defective_question` 与 `defect_reason` 供溯源），并清理 `1M/6 TR[1]` 元泄漏文本 | **4 题** |
| **`answer` / `calculation` 与 `rubric` 自相矛盾** | • `10M/5 MSR[0]`：`reasoning_steps` 写 `60+50+50=160 minutes`，核查原对话（`id=74` 1小时 + `id=3276` 50分钟，`id=3277` 实为助手回复）真实总时长为 **110 分钟**，而 `answer` 与 `rubric` 却要求 `350 minutes`；<br>• `10M/3 TR[1]`（`11` vs `12 days`）；`500K/19 TR[0]`（`23` vs `9 days`）；`500K/9 TR[1]`（`130` vs `170 hours`）；`500K/16 TR[1]`（`21` vs `11 days`）；`500K/22 TR[0]`（`30` vs `33 months`）；`100K/1 TR[0]`（`4` vs `8 weeks`）；`100K/14 TR[0]`（`11` vs `17 days`）；`1M/22 TR[0]`（`13` vs `12 days`）；`1M/24 TR[0]`（Rubric 将 `April 27` 误写为 `April 17`）；`100K/12 TR[1]`（原文明确说截至5月31日写了 `58 days`，Rubric 却错推起点为 `April 5`）。 | 逐题回查原对话轮次与公历算式，统一对齐 `answer`、`calculation_required`、`reasoning_steps` 与 `rubric` | **11 题** |
| **日历跳月与月份天数幻觉** | • `100K/17 TR[1]`：将 `April 20` 至 `July 5`（实为 **76 天**）漏算整整一个月写成 `46 days`；<br>• `500K/34 TR[0]`：把 3 月当成 28 天，将 `March 20` 至 `April 14`（25 天 = **3 周 4 天**）写成 `3 weeks and 1 day`；<br>• `500K/12 TR[1]`：将 `March 22` 至 `April 12`（21 天）误算为 `20 days`。 | 按真实公历天数修正答案、算式与 `rubric` | **3 题** |
| **2024 闰年 2 月 29 日系统性漏算** | 全部跨越 2024 年 2 月的天数差题目（`1M/12 TR[0]` `123→124`、`1M/18 TR[1]` `46→47`、`10M/5 TR[1]` `27→28`、`500K/23 TR[0]` `48→49`、`1M/34 TR[0]` `136→137`）均按平年 28 天计算，系统性少算 1 天。 | 计入 2024 年 2 月 29 日，统一修正答案、算式与 `rubric` | **5 题** |
| **`event_ordering` 元数据计数错位** | 其余 8 处 `total_mentions != len(rubric)`（`100K/14`、`500K/15,22,29`、`1M/1,7,30,32`、`10M/1`）。 | 对齐 `total_mentions` 与题干项数约束 | **8 题** |

> **115 道修复题按能力维度分布**：`contradiction_resolution`（**36/200**，18.0%）、`knowledge_update`（**33/200**，16.5%）、`temporal_reasoning`（**22/200**，11.0%）、`summarization`（**12/200**，6.0%）、`event_ordering`（**9/200**，4.5%）、`preference_following`（**2/200**，1.0%）、`multi_session_reasoning`（**1/200**，0.5%）。

---

### 2. `prompts.py` & `compute_metrics.py` → `fixed/prompts.py`, `fixed/compute_metrics.py`
* **裁判提示词漏传问题文本 `<question>`（致命评测漏洞）**：原 `unified_llm_judge_base_prompt` 要求裁判判断模型回复是否准确回应了 `the QUESTION`，但模板中只有 `- RESPONSE` 与 `- RUBRIC` 占位符，从未注入 `probing_question`；已在 `fixed/prompts.py` 中补齐 `- QUESTION: <question>`，并在 `fixed/compute_metrics.py` 全部 10 个 `evaluate_*` 函数中完成替换注入。
* **`event_ordering` 切分脆弱性与零除保护**：兼容单行编号列表（`1) ... 2) ...`）与多行列表切分，并对空 `rubric` 增加零除保护。

### 3. `run_evaluation.py` & `long_term_memory_methods.py`
* **`run_evaluation.py`**：将 `initialize_models()` 移出每道题的内层循环，避免评测时重复初始化加载 3 个 SentenceTransformer 模型达 2,000 次。
* **`long_term_memory_methods.py`**：原代码 `messages = saved_messages` 使用列表引用别名而非拷贝，导致 `.append()` 原地污染同一会话后续第 2～20 题的历史上下文；已修正为 `messages = list(saved_messages)`。
