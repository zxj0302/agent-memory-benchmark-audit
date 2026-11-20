# MemoryAgentBench 错误与纠正汇总

> **语言切换**: [English](./FIX_SUMMARY.md) | **简体中文**  
> **返回主目录**: [`README_ZH.md`](../README_ZH.md) · [`ALL_FIXES_SUMMARY_ZH.md`](../ALL_FIXES_SUMMARY_ZH.md)

---

## 一、目录与文件对照（1:1 镜像结构）

| 原文件路径 (`original/`) | 修正后路径 (`fixed/`) | 文件说明 |
| :--- | :--- | :--- |
| [`original/Accurate_Retrieval-00000-of-00001.parquet`](./original/Accurate_Retrieval-00000-of-00001.parquet) | [`fixed/Accurate_Retrieval-00000-of-00001.parquet`](./fixed/Accurate_Retrieval-00000-of-00001.parquet) | 精确检索子集（22 行 / 2,000 题） |
| [`original/Conflict_Resolution-00000-of-00001.parquet`](./original/Conflict_Resolution-00000-of-00001.parquet) | [`fixed/Conflict_Resolution-00000-of-00001.parquet`](./fixed/Conflict_Resolution-00000-of-00001.parquet) | 冲突消解子集（8 行 / 800 题） |
| [`original/Long_Range_Understanding-00000-of-00001.parquet`](./original/Long_Range_Understanding-00000-of-00001.parquet) | [`fixed/Long_Range_Understanding-00000-of-00001.parquet`](./fixed/Long_Range_Understanding-00000-of-00001.parquet) | 长程理解与摘要子集（去重后 `110 → 109` 行） |
| [`original/Test_Time_Learning-00000-of-00001.parquet`](./original/Test_Time_Learning-00000-of-00001.parquet) | [`fixed/Test_Time_Learning-00000-of-00001.parquet`](./fixed/Test_Time_Learning-00000-of-00001.parquet) | 测试时上下文学习与推荐子集（6 行 / 700 题） |
| [`original/eval_other_utils.py`](./original/eval_other_utils.py) | [`fixed/eval_other_utils.py`](./fixed/eval_other_utils.py) | 输出解析、分块与词汇评测指标工具函数 |
| [`original/longmem_qa_evaluate.py`](./original/longmem_qa_evaluate.py) | [`fixed/longmem_qa_evaluate.py`](./fixed/longmem_qa_evaluate.py) | LLM-as-a-Judge 问答评测脚本 |
| [`original/graph_rag.py`](./original/graph_rag.py) | [`fixed/graph_rag.py`](./fixed/graph_rag.py) | GraphRAG 记忆方法实现 |
| [`original/agent.py`](./original/agent.py) | [`fixed/agent.py`](./fixed/agent.py) | 统一 Agent 封装与长上下文截断调度器 |

---

## 二、逐文件错误与纠正明细

### 1. 四个 Parquet 数据集子集修复
* **`Conflict_Resolution-00000-of-00001.parquet`**：
  * **新旧事实标准答案颠倒（7 题，严重 Ground-Truth 错误）**：任务模板明确要求“事实冲突时以序号最大（最新）的事实为准”，但 `factconsolidation_sh_32k`（Q8 `London`→`Washington, D.C.`、Q9 `Rome`→`Watertown`）、`factconsolidation_sh_64k`（Q18 `Tom Clancy`→`Torquato Tasso`、Q20 `Europe`→`Asia`）与 `factconsolidation_sh_262k`（Q8 `Kūkai`→`William Waynflete`、Q49 `Providence`→`Ottawa`、Q80 `Riyadh Region`→`London`）共 **7 道题**误将早期旧事实标为标准答案；已全部修正为最新事实。
  * **合成模板拼写损坏修复**：修复全部 **8/8 行**上下文中的高频模板拼写错误（`1,296` 处 `'univeristy'` → `'university'`，`942` 处 `'origianl'` → `'original'`）。
* **`Accurate_Retrieval-00000-of-00001.parquet`**：
  * **Metadata 空值规范化**：17/22 行（除 `longmemeval_s*` 外）的 `metadata["question_ids"]` 为 `None`，遍历时易引发 `TypeError`，已统一规范化为空列表 `[]`。
* **`Long_Range_Understanding-00000-of-00001.parquet`**：
  * **移除完全重复样本行**：`infbench_sum_eng_shots2` 中第 44 行与第 5 行的长文本 `context`（632,149 字符）及标准答案字节级完全一致，已剔除重复行（`110 → 109` 行）。
* **`Test_Time_Learning-00000-of-00001.parquet`**：
  * **清除训练-测试集数据泄漏与高达 53% 的重复 Few-shot 样本**：原 `icl_nlu`（Q96 `"start the vacuum cleaner"`）与 `icl_trec_coarse` / `icl_trec_fine`（Q41 `"What does CPR stand for ?"`）的测试问题连同标准答案直接原样出现在上下文演示样本中（共泄漏 15 次），且 `icl_trec_fine` 等子集有高达 53.0% 的重复演示块；已在上下文中剔除全部测试集泄漏块并对重复演示块去重（共清理 **5,920 个**冗余/泄漏块）。

---

### 2. 评测与基线代码修复
* **`eval_other_utils.py`**：
  * **`substring_exact_match_score` 无词边界假阳性**：原代码 `normalize_answer(gt) in normalize_answer(pred)` 会将拒答 `"I do not know"` 判为命中标准答案 `"no"`、将 `"10"` 判为命中 `"1"`（影响 162 道单数字标签题及多道是非题）；已改为带词边界保护的正则匹配 `(?<!\w)...(?!\w)`。
  * **`parse_output` 首行截断与换行丢失**：原代码在模型输出 `Answer:\n<内容>` 时返回空字符串 `""`，在缺少 `Answer:` 前缀时静默截断掉第一行之后的所有内容；已改用 `re.escape(answer_prefix)` 与 `re.DOTALL` 多行匹配。
  * **`chunk_text_into_sentences` 首句超长产空块**：当首句超过 `chunk_size` 时原逻辑会先 `append("")` 产生空块，已加非空检查修复。
* **`longmem_qa_evaluate.py`**：同步修复继承自 LongMemEval 的 `±1` 时间误差宽容、`knowledge-update` 旧事实混同及 `'yes' in response.lower()` 子串误判。
* **`graph_rag.py`**：原代码第 552、604 行误写 `encoded[127000:]`（丢弃前 127K 正文、只保留超长尾部残片！），已修正为 `encoded[:127000]`。
* **`agent.py`**：原 `_query_long_context_agent` 中 `if self.input_length_limit <= self.context_max_length + 50000:` 条件在大窗口配置下会直接跳过 `_truncate_context_if_needed`，导致超限报错；已修正为无条件执行上下文长度安全截断。
