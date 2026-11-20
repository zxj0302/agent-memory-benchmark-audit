# LongMemEval 错误与纠正汇总

## 一、目录与文件对照（1:1 镜像结构）

| 原文件路径 (`original/`) | 修正后路径 (`fixed/`) | 文件说明 |
| :--- | :--- | :--- |
| [`original/longmemeval_oracle.json`](./original/longmemeval_oracle.json) | [`fixed/longmemeval_oracle.json`](./fixed/longmemeval_oracle.json) | LongMemEval Oracle 子集（500 题，仅含证据会话） |
| [`original/longmemeval_s_cleaned.json.gz`](./original/longmemeval_s_cleaned.json.gz) | [`fixed/longmemeval_s_cleaned.json.gz`](./fixed/longmemeval_s_cleaned.json.gz) | LongMemEval-S Cleaned 完整约 115K-token 长程对话集（500 题，Gzip 无损压缩以适配 GitHub `<100MB` 单文件限制，解压后即标准 `.json`） |
| [`original/evaluate_qa.py`](./original/evaluate_qa.py) | [`fixed/evaluate_qa.py`](./fixed/evaluate_qa.py) | 官方 LLM-as-a-Judge 问答评测脚本 |
| [`original/print_qa_metrics.py`](./original/print_qa_metrics.py) | [`fixed/print_qa_metrics.py`](./fixed/print_qa_metrics.py) | 官方分题型准确率汇总脚本 |

---

## 二、逐文件错误与纠正明细

### 1. `longmemeval_s_cleaned.json.gz` & `longmemeval_oracle.json`

| 错误类别 | 原文件具体问题 | 纠正方式 | 修复统计 |
| :--- | :--- | :--- | :--- |
| **重复 `haystack_session_ids` 导致字典覆盖** | `longmemeval_s_cleaned` 中有 **13 道题**在同一题的 `haystack_session_ids` 列表内重复出现完全相同的 `session_id`（内容相同但分配了不同日期），导致按 `session_id` 构建检索索引的记忆系统静默丢弃前一条会话。 | 为同题内重复出现的会话 ID 追加 `__dup1` 消歧后缀 | `s_cleaned` **13 题** |
| **会话未按时间顺序排列（违背官方文档声明）** | 官方文档声明长对话已按时间戳升序排列，但实测 `s_cleaned` 中 **211/500 题（42.2%）**、`oracle` 中 **34/500 题（6.8%）** 的 `haystack_dates` 存在时间乱序，破坏流式记忆写入与状态更新评测。 | 按解析后的真实时间戳对 `(haystack_dates, haystack_session_ids, haystack_sessions)` 同步稳定升序重排 | `s_cleaned` **211 题**<br>`oracle` **34 题** |
| **`question_date` 早于历史会话的因果倒挂** | `s_cleaned` 中 **76/500 题（15.2%）**、`oracle` 中 **43/500 题（8.6%）** 的提问时间 `question_date` 竟然**早于**其历史对话中最晚会话的时间（“在未来对话发生之前提问”）。 | 将倒挂题目的 `question_date` 钳位修正至最晚历史会话时间，并将原时间戳保留在 `audit_orig_question_date` | `s_cleaned` **76 题**<br>`oracle` **43 题** |
| **答案字段数据类型不统一** | 两个子集中均有 **32 道题**的 `answer` 为整型 `int`（如 `3`、`2`）而非字符串 `str`。 | 统一转换为 `str` | 各 **32 题** |

---

### 2. `original/evaluate_qa.py` → `fixed/evaluate_qa.py`
* **移除时间推理题过度宽容规则**：原 `temporal-reasoning` 提示词规定模型算错 `±1` 天/周/月仍判全对（*"off-by-one errors for the number of days/weeks/months are acceptable"*），掩盖了日期边界计算缺陷；已修正为要求精确时间数值匹配。
* **收紧 `knowledge-update` 旧事实混同规则**：原提示词规定只要回答里顺带提到了更新后的答案，即使同时把已作废的旧状态当成当前事实输出也判对；已修正为明确拒绝将过时信息当作当前有效状态的回答。
* **修复 `'yes' in eval_response.lower()` 子串假阳性**：防止裁判输出如 *"No, saying yes here would be wrong"* 被误判为正确，改为严格的词首匹配 `startswith('yes')`。

### 3. `original/print_qa_metrics.py` → `fixed/print_qa_metrics.py`
* **移除硬编码裁判模型断言与空列表异常**：原脚本第 20 行硬编码 `assert entry['autoeval_label']['model'] == 'gpt-4o-2024-08-06'`，更换任何裁判模型版本即崩溃，且在子集评测某题型样本为空时会产生 `NaN`；已移除硬编码断言并增加非空保护。
