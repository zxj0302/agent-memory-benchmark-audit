# LongMemEval — File-by-File Error & Fix Summary

> **Language**: **English** | [简体中文](./FIX_SUMMARY_ZH.md)  
> **Back to Root**: [`README.md`](../README.md) · [`ALL_FIXES_SUMMARY.md`](../ALL_FIXES_SUMMARY.md)

---

## 1. Directory & 1:1 Mirrored File Mapping

| Original File (`original/`) | Corrected File (`fixed/`) | Description |
| :--- | :--- | :--- |
| [`original/longmemeval_oracle.json`](./original/longmemeval_oracle.json) | [`fixed/longmemeval_oracle.json`](./fixed/longmemeval_oracle.json) | LongMemEval Oracle subset (500 QAs containing evidence sessions only) |
| [`original/longmemeval_s_cleaned.json.gz`](./original/longmemeval_s_cleaned.json.gz) | [`fixed/longmemeval_s_cleaned.json.gz`](./fixed/longmemeval_s_cleaned.json.gz) | LongMemEval-S Cleaned (~115K-token haystack, 500 QAs; lossless Gzip to fit GitHub `<100 MB` file limit) |
| [`original/evaluate_qa.py`](./original/evaluate_qa.py) | [`fixed/evaluate_qa.py`](./fixed/evaluate_qa.py) | Official LLM-as-a-Judge QA evaluation script |
| [`original/print_qa_metrics.py`](./original/print_qa_metrics.py) | [`fixed/print_qa_metrics.py`](./fixed/print_qa_metrics.py) | Official per-category accuracy aggregation script |

---

## 2. Detailed Errors & Fixes by File

### 2.1 `longmemeval_s_cleaned.json.gz` & `longmemeval_oracle.json`

| Defect Category | Exact Issue in Original Files | Fix Applied | Fixed Count |
| :--- | :--- | :--- | :--- |
| **Duplicate `haystack_session_ids` Causing Silent Index Overwrites** | In `longmemeval_s_cleaned`, **13 items** repeat an identical `session_id` twice within the same question's `haystack_session_ids` (identical turns paired with different timestamps), causing memory systems keyed by `session_id` to silently overwrite the earlier session. | Disambiguated duplicate IDs within each question by appending `__dup1` | `s_cleaned`: **13 QAs** |
| **Unsorted Haystack Sessions (Violating Official Specification)** | Although the official documentation states sessions are chronologically ordered, **211/500 items (42.2%)** in `s_cleaned` and **34/500 items (6.8%)** in `oracle` have out-of-order `haystack_dates`, corrupting streaming memory ingestion and state-update evaluation. | Co-sorted `(haystack_dates, haystack_session_ids, haystack_sessions)` in stable chronological order | `s_cleaned`: **211 QAs**<br>`oracle`: **34 QAs** |
| **Causal Timestamp Inversion (`question_date < max(haystack_dates)`)** | In **76/500 items (15.2%)** of `s_cleaned` and **43/500 items (8.6%)** of `oracle`, the query timestamp `question_date` is **earlier** than the latest session in the user's conversation history (asking about events before they happened). | Clamped `question_date` to `max(haystack_dates)` and preserved the original timestamp in `audit_orig_question_date` | `s_cleaned`: **76 QAs**<br>`oracle`: **43 QAs** |
| **Non-String Gold Answer Types** | **32 items** in both subsets stored numeric `answer` values as `int` (`3`, `2`) instead of `str`. | Normalized all `answer` values to `str` | **32 QAs** each |

---

### 2.2 `original/evaluate_qa.py` → `fixed/evaluate_qa.py`
* **Removed Over-Permissive Off-by-One Temporal Grading**: Removed the `temporal-reasoning` prompt clause instructing the judge that *"off-by-one errors for the number of days/weeks/months are acceptable"*, restoring exact temporal evaluation.
* **Tightened `knowledge-update` Stale-Fact Grading**: Prevented responses that output superseded/outdated facts alongside updated facts as current truth from being graded `yes`.
* **Fixed `'yes' in eval_response.lower()` Substring False Positive**: Replaced substring matching (which misclassified *"No, saying yes would be incorrect"* as `True`) with `eval_response.strip().lower().startswith('yes')`.

### 2.3 `original/print_qa_metrics.py` → `fixed/print_qa_metrics.py`
* **Removed Hardcoded Judge Model Assertion & Empty-Slice `NaN`**: Removed `assert entry['autoeval_label']['model'] == 'gpt-4o-2024-08-06'` so alternative judge models work out of the box, and guarded empty category slices against `NaN` output.
