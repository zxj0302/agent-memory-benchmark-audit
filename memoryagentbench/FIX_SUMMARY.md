# MemoryAgentBench — File-by-File Error & Fix Summary

> **Language**: **English** | [简体中文](./FIX_SUMMARY_ZH.md)  
> **Back to Root**: [`README.md`](../README.md) · [`ALL_FIXES_SUMMARY.md`](../ALL_FIXES_SUMMARY.md)

---

## 1. Directory & 1:1 Mirrored File Mapping

| Original File (`original/`) | Corrected File (`fixed/`) | Description |
| :--- | :--- | :--- |
| [`original/Accurate_Retrieval-00000-of-00001.parquet`](./original/Accurate_Retrieval-00000-of-00001.parquet) | [`fixed/Accurate_Retrieval-00000-of-00001.parquet`](./fixed/Accurate_Retrieval-00000-of-00001.parquet) | Accurate Retrieval split (22 rows / 2,000 QAs) |
| [`original/Conflict_Resolution-00000-of-00001.parquet`](./original/Conflict_Resolution-00000-of-00001.parquet) | [`fixed/Conflict_Resolution-00000-of-00001.parquet`](./fixed/Conflict_Resolution-00000-of-00001.parquet) | Conflict Resolution split (8 rows / 800 QAs) |
| [`original/Long_Range_Understanding-00000-of-00001.parquet`](./original/Long_Range_Understanding-00000-of-00001.parquet) | [`fixed/Long_Range_Understanding-00000-of-00001.parquet`](./fixed/Long_Range_Understanding-00000-of-00001.parquet) | Long-Range Understanding & Summarization split (`110 → 109` deduplicated rows) |
| [`original/Test_Time_Learning-00000-of-00001.parquet`](./original/Test_Time_Learning-00000-of-00001.parquet) | [`fixed/Test_Time_Learning-00000-of-00001.parquet`](./fixed/Test_Time_Learning-00000-of-00001.parquet) | Test-Time Learning & Recommendation split (6 rows / 700 QAs) |
| [`original/eval_other_utils.py`](./original/eval_other_utils.py) | [`fixed/eval_other_utils.py`](./fixed/eval_other_utils.py) | Output parser, sentence chunker, and lexical scoring utilities |
| [`original/longmem_qa_evaluate.py`](./original/longmem_qa_evaluate.py) | [`fixed/longmem_qa_evaluate.py`](./fixed/longmem_qa_evaluate.py) | LLM-as-a-Judge QA evaluation script |
| [`original/graph_rag.py`](./original/graph_rag.py) | [`fixed/graph_rag.py`](./fixed/graph_rag.py) | GraphRAG memory baseline module |
| [`original/agent.py`](./original/agent.py) | [`fixed/agent.py`](./fixed/agent.py) | Unified Agent wrapper & long-context truncation dispatcher |

---

## 2. Detailed Errors & Fixes by File

### 2.1 Parquet Dataset Splits (`4 Splits`)
* **`Conflict_Resolution-00000-of-00001.parquet`**:
  * **Inverted Conflict Ground-Truth Answers (7 QAs, Critical Semantic Bug)**: Although task instructions explicitly state that the newest fact (largest serial number) supersedes earlier facts, **7 QAs** across `factconsolidation_sh_32k` (Q8 `London`→`Washington, D.C.`, Q9 `Rome`→`Watertown`), `factconsolidation_sh_64k` (Q18 `Tom Clancy`→`Torquato Tasso`, Q20 `Europe`→`Asia`), and `factconsolidation_sh_262k` (Q8 `Kūkai`→`William Waynflete`, Q49 `Providence`→`Ottawa`, Q80 `Riyadh Region`→`London`) labeled the **outdated** fact as Gold. Updated all 7 answers to the newest fact.
  * **Synthetic Template Typo Repair**: Fixed systematic spelling corruption across all **8/8 rows** (`1,296` occurrences of `'univeristy'` → `'university'`, `942` occurrences of `'origianl'` → `'original'`).
* **`Accurate_Retrieval-00000-of-00001.parquet`**:
  * **Null Metadata Normalization**: Normalized `metadata["question_ids"] = None` in 17/22 rows (all non-`longmemeval_s*` rows) to `[]` to prevent `TypeError` on iteration.
* **`Long_Range_Understanding-00000-of-00001.parquet`**:
  * **Exact Duplicate Row Removal**: Removed row `44` (`infbench_sum_eng_shots2`), whose 632,149-character `context` and gold summary were byte-for-byte identical to row `5` (`110 → 109` rows).
* **`Test_Time_Learning-00000-of-00001.parquet`**:
  * **Train-Test Data Leakage & Duplicate ICL Shot Cleanup**: Removed verbatim test queries + gold labels leaked inside the demonstration context (`icl_nlu` Q96 `"start the vacuum cleaner"`, `icl_trec_coarse`/`icl_trec_fine` Q41 `"What does CPR stand for ?"`, leaked 15 times total) and deduplicated repeated demonstration blocks (up to 53.0% duplication in `icl_trec_fine`), cleaning **5,920 redundant/leaked blocks** in total.

---

### 2.2 Evaluation & Baseline Code (`4 Files`)
* **`eval_other_utils.py`**:
  * **`substring_exact_match_score` Word-Boundary Bug**: Replaced raw `normalize_answer(gt) in normalize_answer(pred)` (which scored `"I do not know"` as matching Gold `"no"` and `"10"` as matching Gold `"1"`, affecting 162 single-digit ICL labels and yes/no items) with word-boundary regex `(?<!\w)...(?!\w)`.
  * **`parse_output` Multiline & Prefix-Escape Fix**: Fixed `re.escape(answer_prefix)` with `re.DOTALL` so `Answer:\n<text>` no longer parses as empty string `""` and multiline outputs without `Answer:` are not silently truncated to line 1.
  * **`chunk_text_into_sentences` Empty First Chunk Bug**: Prevented emitting `""` as chunk 0 when the first sentence exceeds `chunk_size`.
* **`longmem_qa_evaluate.py`**: Synchronized judge fixes from LongMemEval (removed `±1` temporal off-by-one leniency, tightened `knowledge-update` stale-fact handling, and replaced `'yes' in response.lower()` with `startswith('yes')`).
* **`graph_rag.py`**: Fixed inverted token-limit slice `encoded[127000:]` (lines 552 & 604, which discarded the first 127K tokens and kept only the overflow tail!) to `encoded[:127000]`.
* **`agent.py`**: Removed the broken `if self.input_length_limit <= self.context_max_length + 50000:` guard in `_query_long_context_agent` that bypassed `_truncate_context_if_needed(tokenizer)` on large-window configurations.
