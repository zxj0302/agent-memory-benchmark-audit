# Agent Memory Benchmark Audit & Corrected Suite

对当前最主流的 **6 个大模型智能体长期记忆评测基准（Agent Memory Benchmarks）**——**LoCoMo（含 LoCoMo_refined）**、**BEAM**、**LongMemEval**、**MemoryAgentBench**、**MemBench** 与 **PerLTQA**——开展多轮全量结构审查、评测源码审计与深度语义 Ground-Truth 核验后构建的**原版对照与修正版基准合集**。

* **全量数据集错误与纠正总表**：[`ALL_FIXES_SUMMARY.md`](./ALL_FIXES_SUMMARY.md)

---

## 一、项目背景与核心发现

随着长上下文模型与外部记忆框架（RAG、Graph Memory、Agentic Memory）的快速发展，学界提出了多个长期记忆评测基准。然而，现有基准普遍依赖长对话多阶段合成与自动化判分（LLM-as-a-Judge 或启发式字符串指标），在缺乏严格确定性校验的情况下引入了大量**系统性标准答案错误、时序因果倒挂与评测代码漏洞**，直接扭曲了不同记忆机制之间的优劣排序：

1. **标准答案与人物归属错位（Ground-Truth Corruption）**：如 `LoCoMo` 中有 13 道题将对话者 A 自述的个人经历张冠李戴到对话者 B 头上，其中 **10 道题在专门清洗数据的 `LoCoMo_refined` 中依然漏检残留**；`MemoryAgentBench` 冲突消解子集中有 **7 道题**违背“最新事实优先”规则，将已被覆盖的早期旧事实标为标准答案。
2. **跨样本评分标准污染与凭空捏造幻觉题（Rubric Contamination & Phantom QA）**：`BEAM` 官方评测脚本仅读取 `rubric` 而不读取 `ideal_answer`，但其 `500K/8`（椭圆/双曲线数学题）与 `500K/9`（等比数列数学题）却原样粘贴了软件工程会话的 `"Cypress tests ... passing rate of 92%"` 评分标准；另有 `500K/2` 在简历解析编程对话中凭空追问全文出现 0 次的“订酒店日期”，`500K/31` 在原对话漏掉日期后竟将内部大纲编号 `1,18` 幻觉成 `18 days`。
3. **日历与算术系统性失真（Systematic Calendar & Math Blindness）**：`BEAM` 中凡跨越 2024 年 2 月的天数计算题（共 5 道），全部按平年 28 天计算而**系统性漏算闰年 2 月 29 日**；另有题目将 `April 20` 至 `July 5`（76 天）漏算整月写成 `46 days`，或出现同一题内推理步骤写 `160` 分钟、真实对话合计 `110` 分钟、而 `rubric` 要求 `350` 分钟的三方冲突。
4. **时序乱序与因果倒挂（Temporal & Causal Inversion）**：`LongMemEval` 官方声明长程历史会话按时间升序排列，但实测 `longmemeval_s_cleaned` 中 **42.2%（211/500）会话乱序**，且 **15.2%（76/500）提问时间 `question_date` 早于历史对话证据发生时间**。
5. **多选退化、跨语言错位与评测器假阳性（Degenerate Options & Broken Evaluators）**：`MemBench` 有 **777 道四选一题**存在字面完全重复的选项，另有 **3,047 处**如 `"eight of week"`、`"three day"` 的模板拼接病句；`PerLTQA` 英文版将角色 `"龙美丽"` 机翻为 `'dragon beautiful'` 导致 265 条问答与记忆库 `'Long Meili'` 完全失联，且检索器训练前向代码误加 `with torch.no_grad():` 锁死梯度；`LoCoMo` 原版用词袋集合相等冒充 EM、用 `rouge-1` 冒充 `rouge-l`；`MemoryAgentBench` 用无词边界子串匹配导致 `"no" in "i do not know"` 与 `"1" in "10"` 被判为正确。

---

## 二、仓库目录组织与架构设计

本仓库采用**零脚本污染、严格 1:1 同名镜像**的清晰架构（不包含任何临时数据处理脚本）：

* **根目录**：包含项目说明 [`README.md`](./README.md) 与所有数据集修复总览 [`ALL_FIXES_SUMMARY.md`](./ALL_FIXES_SUMMARY.md)。
* **6 个独立数据集子文件夹**：每个子文件夹均包含且仅包含三部分：
  1. `original/`：未修改的原始数据集文件与原始被审评测/加载代码；
  2. `fixed/`：修正后的干净数据集文件与修复版代码，**相对路径与文件名同 `original/` 保持 100% 一一对应**，既可直接 `diff` 审查每一行改动，也可直接覆盖回官方仓库即插即用；
  3. `FIX_SUMMARY.md`：该数据集内每个文件的错误定位、成因分析与纠正统计。

```text
agent-memory-benchmark-audit/
├── README.md                                   # 项目说明、架构设计、审计方法论与使用指南
├── ALL_FIXES_SUMMARY.md                        # 六大数据集所有错误与纠正汇总合集
│
├── locomo/                                     # [1] LoCoMo & LoCoMo_refined (146 对照文件总计: 4对)
│   ├── FIX_SUMMARY.md                          #     逐文件错误与纠正明细
│   ├── original/                               #     locomo10.json, locomo_refined.json, evaluation.py, llm_judge.py
│   └── fixed/                                  #     1:1 同名修正版数据与评测代码
│
├── beam/                                       # [2] BEAM (100K / 500K / 1M / 10M, 104对镜像文件)
│   ├── FIX_SUMMARY.md                          #     逐文件错误与纠正明细
│   ├── original/                               #     probing_questions/{100K,500K,1M,10M}/*/probing_questions.json (100个)
│   │                                           #     + prompts.py, compute_metrics.py, run_evaluation.py, long_term_memory_methods.py
│   └── fixed/                                  #     全量 100 组修正后探测题集（含 30 组缺陷修复）+ 4 个修复版源码
│
├── longmemeval/                                # [3] LongMemEval (Oracle + S_cleaned, 4对镜像文件)
│   ├── FIX_SUMMARY.md                          #     逐文件错误与纠正明细
│   ├── original/                               #     longmemeval_oracle.json, longmemeval_s_cleaned.json.gz, evaluate_qa.py, print_qa_metrics.py
│   └── fixed/                                  #     1:1 同名修正版数据与评测代码
│
├── memoryagentbench/                           # [4] MemoryAgentBench (4大能力维度 Parquet + 核心源码, 8对镜像文件)
│   ├── FIX_SUMMARY.md                          #     逐文件错误与纠正明细
│   ├── original/                               #     Accurate_Retrieval / Conflict_Resolution / Long_Range_Understanding /
│   │                                           #     Test_Time_Learning (.parquet) + eval_other_utils.py, longmem_qa_evaluate.py, graph_rag.py, agent.py
│   └── fixed/                                  #     1:1 同名修正版 Parquet 与评测/Agent 源码
│
├── membench/                                   # [5] MemBench (FirstAgent 11子集 + ThirdAgent 8子集 + 框架源码, 21对镜像文件)
│   ├── FIX_SUMMARY.md                          #     逐文件错误与纠正明细
│   ├── original/                               #     FirstAgent/*.json (11个), ThirdAgent/*.json (8个), MembenchAgent.py, CommonMemory.py
│   └── fixed/                                  #     1:1 同名修正版 JSON 与框架代码
│
└── perltqa/                                    # [6] PerLTQA (角色长期记忆库 + 问答集 + 检索训练源码, 5对镜像文件)
    ├── FIX_SUMMARY.md                          #     逐文件错误与纠正明细
    ├── original/                               #     perltmem_en_v2.json, perltqa_en_v2.json, dataset.py, modeling.py, encoder_corpus.py
    └── fixed/                                  #     1:1 同名修正版记忆库、问答集与加载/训练代码
```

---

## 三、审计方法论：三层混合审计漏斗（3-Layer Hybrid Audit Funnel）

面对跨越数千万 tokens 与数万道题目的长程对话语料，单纯依赖人工/模型从头通读完整对话会因低信噪比产生注意力稀释（*Lost-in-the-Middle*），而单纯依赖 Schema 脚本又无法识别语义张冠李戴与数学错算。本项目通过三层递进机制确保高召回率与零误修：

1. **第一层：全量确定性不变量校验与算术/日历求解器**
   * 对全部数据集执行字段完整性、类型一致性、引用指针存在性、时序单调性、多选选项唯一性及训练-测试集交集扫描；
   * 使用确定性公历日期求解器与算术解析器全量重算日期跨度（含闰年边界）、相对星期换算、多会话时长累加与事实更新覆盖链；
   * 全量交叉比对 `answer` / `calculation_required` / `reasoning_steps` 与 `rubric` 的数值一致性，检测跨会话重复粘贴的 `rubric` 以及在原对话中实词零命中的幻觉问题；
   * 逐行审查官方评测器与记忆基线源码，定位指标计算、Prompt 占位符注入、上下文截断与状态共享缺陷。
2. **第二层：证据邻域局部切片（Evidence-Window Slicing）**
   * 根据每道题的证据锚点（`evidence`、`source_chat_ids`、`target_step_id`、`Reference Memory`），精准提取目标轮次及前后 `±2` 轮局部对话切片，剥离 99% 无关背景噪声，专门核验双人对话中的第一人称主语归属、实体指代与事实存在性。
3. **第三层：高危候选原文回溯与精读定性**
   * 对前两层检出的全部异常候选逐条回查原始会话与生成大纲，剔除跨轮指代（如前一轮展示海报、后一轮点明片名）或特殊索引约定造成的启发式假阳性，确保写入 `fixed/` 的每一项修正均有确凿原文依据。

---

## 四、六大数据集评测偏向与修正统计一览

| 基准名称 | 核心评测偏向 | 数据层主要缺陷与修复 | 评测/基线代码主要缺陷与修复 | 详情文档 |
| :--- | :--- | :--- | :--- | :--- |
| **LoCoMo & LoCoMo_refined** | 双人长周期多模态社交对话记忆（单跳、多跳、时间、开放域、对抗拒答） | 修复原版 **13 题**（精炼版残留 **10 题**）说话人张冠李戴、**3 题**星期错算/笔误、**8 处**证据轮次偏移、**16 处**证据格式损坏与越界指针、Cat 5 答案泄漏与 `int` 类型答案 | 修复 `evaluation.py` 词袋无序 EM、误删 `"and"`、`rouge-1` 冒充 `rouge-l`、多跳单向 F1；消除 `llm_judge.py` 额外细节判分自相矛盾 | [`locomo/FIX_SUMMARY.md`](./locomo/FIX_SUMMARY.md) |
| **BEAM** | 100K–10M 超长多会话十维综合能力，完全依赖 Rubric LLM-Judge | 修正 **30 个会话文件（34 道题）**：修复 **2 题**跨会话 `"Cypress tests"` Rubric 污染、**3 题**凭空捏造或将大纲编号 `1,18` 当天数的幻觉题、**11 题**答案与 Rubric 数值冲突、**3 题**跳月错算、**5 题** 2024 闰年 2 月 29 日漏算、**10 题**事件排序计数错位 | 修复 `prompts.py` + `compute_metrics.py` 裁判漏传 `<question>` 占位符、`run_evaluation.py` 循环内重复加载模型 2,000 次、`long_term_memory_methods.py` 引用别名跨题污染 | [`beam/FIX_SUMMARY.md`](./beam/FIX_SUMMARY.md) |
| **LongMemEval** | 多会话用户体验式长期记忆（信息抽取、多会话聚合、时间推理、知识更新、拒答） | 修复 `s_cleaned` **13 处**重复 `session_id`、**211 题**（`oracle` **34 题**）历史会话时间乱序、**76 题**（`oracle` **43 题**）`question_date` 早于历史会话的因果倒挂、**32 题** `int` 答案 | 移除 `evaluate_qa.py` 对 `±1` 天/周/月算术错误的宽容、收紧旧事实混同判分、修复 `'yes'` 子串假阳性；移除 `print_qa_metrics.py` 硬编码裁判模型断言 | [`longmemeval/FIX_SUMMARY.md`](./longmemeval/FIX_SUMMARY.md) |
| **MemoryAgentBench** | 面向四类核心记忆能力（精确检索、测试时学习、长程理解、冲突消解）的多源整合基准 | 修复 `Conflict_Resolution` **7 题**新旧冲突标准答案颠倒与 2,238 处模板拼写损坏；剔除 `Long_Range_Understanding` 完全重复样本；清理 `Test_Time_Learning` 测试集泄漏与 **5,920 个**重复/泄漏演示块；规范化 `Accurate_Retrieval` 空值 | 修复 `eval_other_utils.py` 无词边界子串假阳性（`"no" in "i do not know"`）、`Answer:\n` 换行解析为空与分句产空块；修复 `graph_rag.py` `encoded[127000:]` 反向截断与 `agent.py` 截断绕过 | [`memoryagentbench/FIX_SUMMARY.md`](./memoryagentbench/FIX_SUMMARY.md) |
| **MemBench** | 第一人称交互（FirstAgent）与第三人称观察（ThirdAgent）双视角结构化多选评测 | 修复全部 19 个 JSON 子集中 **3,047 处**模板拼接病句答案/选项（如 `"eight of week"`→`"eight weeks"`）、消歧 **777 道题（787 槽位）**重复选项、规范化 **7,460 个** `list` 选项与 **65 条** `None` 消息 | 修复 `MembenchAgent.py` 与 `CommonMemory.py` 中在 Python 3 下必崩的 `raise "..."` 字符串异常语法 | [`membench/FIX_SUMMARY.md`](./membench/FIX_SUMMARY.md) |
| **PerLTQA** | 角色画像、社会关系、事件与对话四层个性化长期记忆检索与问答 | 修复 `perltmem_en_v2` **25 名**角色字符串化社会关系、修正机翻错位角色名 `'dragon beautiful'`→`'Long Meili'`（**265 处**）、反序列化 **8,236 题** `Reference Memory`、重算恢复 **11,487 个**英文字符级 `Memory Anchors`、剔除 **3 题**背景缺失不可答题 | 修复 `dataset.py` 硬编码中文键名在英文集上崩溃、修复 `modeling.py` `with torch.no_grad():` 阻断检索器训练梯度、修复 `encoder_corpus.py` `torch.cuda.is_available` 漏写括号崩溃 | [`perltqa/FIX_SUMMARY.md`](./perltqa/FIX_SUMMARY.md) |

---

## 五、使用说明与差异核验（Usage & Verification）

### 1. 直接替换用于公平评测
由于每个子目录下的 `fixed/` 与 `original/` 保持严格的相对路径镜像，您可以直接将 `fixed/` 下的文件覆盖至对应官方仓库的相应目录：

* **LongMemEval 大文件解压**：为满足 GitHub 单文件 `< 100 MB` 限制，`longmemeval_s_cleaned` 采用无损 Gzip 存储，使用前解压即可得到标准 JSON：
  ```bash
  gunzip -k longmemeval/fixed/longmemeval_s_cleaned.json.gz
  # 输出: longmemeval/fixed/longmemeval_s_cleaned.json
  ```
* **BEAM 探测题整目录替换**：下载 [BEAM 官方背景对话 `chats/`](https://github.com/mohammadtavakoli78/BEAM) 后，可用 `beam/fixed/probing_questions/<split>/<id>/probing_questions.json` 覆盖对应会话的 `probing_questions/probing_questions.json`，并用 `beam/fixed/*.py` 替换 `src/` 下对应评测源码。

### 2. 快速审查任意文件的修改前后差异（Diff）
* **对比评测代码或 JSON 文本差异**：
  ```bash
  diff -u locomo/original/evaluation.py locomo/fixed/evaluation.py
  diff -u beam/original/probing_questions/500K/8/probing_questions.json \
          beam/fixed/probing_questions/500K/8/probing_questions.json
  ```
* **在 Python 中加载对比 Parquet 数据集（以 MemoryAgentBench 为例）**：
  ```python
  import pyarrow.parquet as pq

  orig = pq.read_table("memoryagentbench/original/Conflict_Resolution-00000-of-00001.parquet")
  fixed = pq.read_table("memoryagentbench/fixed/Conflict_Resolution-00000-of-00001.parquet")
  print("Rows:", orig.num_rows, "->", fixed.num_rows)
  ```

---

## 六、许可证与致谢（License & Provenance）

本仓库收录的 `original/` 原始数据集与评测代码片段版权归各原基准作者所有（分别遵循原仓库开源协议：LoCoMo、LoCoMo_refined、BEAM、LongMemEval、MemoryAgentBench、MemBench、PerLTQA）。本仓库提供的审计报告、错误台账与修正补丁仅供学术研究与可复现评测基准改进使用。
