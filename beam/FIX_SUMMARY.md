# BEAM — File-by-File Error & Fix Summary

> **Language**: **English** | [简体中文](./FIX_SUMMARY_ZH.md)  
> **Back to Root**: [`README.md`](../README.md) · [`ALL_FIXES_SUMMARY.md`](../ALL_FIXES_SUMMARY.md)

---

## 1. Directory & 1:1 Mirrored File Mapping

| Original File (`original/`) | Corrected File (`fixed/`) | Description |
| :--- | :--- | :--- |
| [`original/probing_questions/`](./original/probing_questions/) | [`fixed/probing_questions/`](./fixed/probing_questions/) | All **100 conversations** (`100K`: 20, `500K`: 35, `1M`: 35, `10M`: 10) totaling 2,000 probing QAs & rubrics (30 defective files fixed, 70 clean files preserved as a complete drop-in directory) |
| [`original/prompts.py`](./original/prompts.py) | [`fixed/prompts.py`](./fixed/prompts.py) | LLM-Judge & synthesis prompt templates |
| [`original/compute_metrics.py`](./original/compute_metrics.py) | [`fixed/compute_metrics.py`](./fixed/compute_metrics.py) | Scoring & LLM-Judge invocation across all 10 memory dimensions |
| [`original/run_evaluation.py`](./original/run_evaluation.py) | [`fixed/run_evaluation.py`](./fixed/run_evaluation.py) | Batch evaluation entry script |
| [`original/long_term_memory_methods.py`](./original/long_term_memory_methods.py) | [`fixed/long_term_memory_methods.py`](./fixed/long_term_memory_methods.py) | Long-term memory baseline inference implementation |

> [!NOTE]
> Unmodified haystack conversation files (`chats/*/*/chat.json`, ~4.1 GB total) were audited read-only as ground-truth reference and can be obtained from the [official BEAM repository](https://github.com/mohammadtavakoli78/BEAM) and paired directly with `fixed/probing_questions/`.

---

## 2. Detailed Errors & Fixes by File

### 2.1 `original/probing_questions/` → `fixed/probing_questions/` (30 Conversation Files, 34 Defective QAs)

Because BEAM's `run_evaluation.py` passes **only `rubric` (ignoring `answer` and `ideal_answer`)** to the LLM Judge, every rubric contamination or arithmetic error below directly penalizes correct model responses:

| Defect Category | Exact Issue in Original Files | Fix Applied | Fixed Count |
| :--- | :--- | :--- | :--- |
| **Cross-Conversation Rubric Copy-Paste Contamination (Critical)** | `500K/8 CR[0]` (ellipse/hyperbola math chat) and `500K/9 CR[0]` (geometric sequence math chat) both copied `1M/2 CR[1]`'s software-testing rubric verbatim: `"LLM response should mention: you said you created Cypress tests for user login and personalized feed with a passing rate of 92%"`. | Rewrote `rubric` to match each conversation's actual math contradiction and `ideal_answer` | **2 QAs** |
| **Fabricated Phantom QAs & Date-Stripped Anchor Hallucinations** | • `500K/2 TR[0]`: Topic is an NLP Resume Analyzer (`"hotel"` appears **0 times** in `chat.json`), yet asks about booking a hotel on April 10;<br>• `1M/7 TR[0]`: Topic is Ordinary Differential Equations, yet asks about non-existent *"project kickoff"* and *"final review meeting"*;<br>• `500K/31 TR[1]`: Dates (`Feb 20` & `Mar 2`, 11 days in `plan.txt`) were omitted from `chat.json`, and the generator hallucinated `"18 days"` from the internal plan index `->-> 1,18`;<br>• `1M/6 TR[1]`: Gold answer leaked internal prompt jargon (`"inferred from batch anchors"`). | Replaced the 3 unanswerable phantom QAs with verified dated temporal QAs from their own `chat.json` (preserving `replaced_defective_question` & `defect_reason` for provenance) and cleaned meta-leakage in `1M/6 TR[1]` | **4 QAs** |
| **Internal Self-Contradictions (`answer`/`calculation` vs. `rubric`)** | • `10M/5 MSR[0]`: `reasoning_steps` says `60+50+50=160 minutes`, actual `chat.json` (`id=74` 1 hr + `id=3276` 50 min; `id=3277` is an assistant reply) sums to **110 minutes**, while `answer` & `rubric` demand **`350 minutes`**;<br>• `10M/3 TR[1]` (`11 days` in answer/calc vs. `12 days` in rubric);<br>• `500K/19 TR[0]` (`23 days` in calc vs. `9 days` in answer/rubric);<br>• `500K/9 TR[1]` (`130` vs. `170 hours`); `500K/16 TR[1]` (`21` vs. `11 days`); `500K/22 TR[0]` (`30` vs. `33 months`); `100K/1 TR[0]` (`4` vs. `8 weeks`); `100K/14 TR[0]` (`11` vs. `17 days`); `1M/22 TR[0]` (`13` vs. `12 days`); `1M/24 TR[0]` (rubric typo `April 17` instead of `April 27`); `100K/12 TR[1]` (rubric hallucinated `April 5` start date for 58-day streak ending `May 31`). | Verified against raw `chat.json` turns and exact arithmetic; reconciled `answer`, `calculation_required`, `reasoning_steps`, and `rubric` | **11 QAs** |
| **Skipped-Month & Month-Length Calendar Hallucinations** | • `100K/17 TR[1]`: `April 20` to `July 5` is **76 days**, miscalculated as `46 days` (skipped a 30-day month);<br>• `500K/34 TR[0]`: `March 20` to `April 14` is **25 days (3 weeks and 4 days)**, miscalculated as `3 weeks and 1 day` (treated March as 28 days);<br>• `500K/12 TR[1]`: `March 22` to `April 12` (21 days) miscalculated as `20 days`. | Updated `answer`, `calculation_required`, and `rubric` to exact calendar durations | **3 QAs** |
| **Systematic 2024 Leap-Year (`Feb 29`) Blindness** | All 5 day-count questions spanning February 2024 (`1M/12 TR[0]` `123→124`, `1M/18 TR[1]` `46→47`, `10M/5 TR[1]` `27→28`, `500K/23 TR[0]` `48→49`, `1M/34 TR[0]` `136→137`) treated Feb 2024 as 28 days, undercounting by 1 day. | Accounted for February 29, 2024 across `answer`, `calculation_required`, and `rubric` | **5 QAs** |
| **`event_ordering` Metadata & Broken Reference Cleanup** | 9 instances where `total_mentions != len(rubric)` (`100K/14`, `500K/15,22,29`, `1M/1,7,30,32`, `10M/1`) plus invalid `'--'` reference in `10M/6` `event_ordering[0]`. | Synced `total_mentions` and removed invalid reference markers | **10 QAs** |

---

### 2.2 `prompts.py` & `compute_metrics.py` → `fixed/prompts.py`, `fixed/compute_metrics.py`
* **Judge Prompt Missing `<question>` Placeholder (Critical Evaluation Bug)**: `unified_llm_judge_base_prompt` instructed the judge to verify whether the response answered `the QUESTION`, but only injected `- RESPONSE` and `- RUBRIC` without `<question>`. Added `- QUESTION: <question>` in `fixed/prompts.py` and injected `.replace("<question>", probing_question)` across all 10 `evaluate_*` functions in `fixed/compute_metrics.py`.
* **Robust `event_ordering` Parsing & Zero-Division Guard**: Supported inline numbered lists (`1) ... 2) ...`) in addition to newline splits, and guarded `score / len(rubric)` against empty rubrics.

### 2.3 `run_evaluation.py` & `long_term_memory_methods.py`
* **`run_evaluation.py`**: Hoisted `initialize_models()` out of the inner per-question loop to prevent reloading 3 SentenceTransformer models 2,000 times.
* **`long_term_memory_methods.py`**: Fixed list aliasing (`messages = saved_messages` → `messages = list(saved_messages)`) so `.append()` no longer mutates and contaminates subsequent questions (Q2–Q20) in the same conversation.
