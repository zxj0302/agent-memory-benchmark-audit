# LoCoMo & LoCoMo_refined — File-by-File Error & Fix Summary

> **Language**: **English** | [简体中文](./FIX_SUMMARY_ZH.md)  
> **Back to Root**: [`README.md`](../README.md) · [`ALL_FIXES_SUMMARY.md`](../ALL_FIXES_SUMMARY.md)

---

## 1. Directory & 1:1 Mirrored File Mapping

| Original File (`original/`) | Corrected File (`fixed/`) | Description |
| :--- | :--- | :--- |
| [`original/locomo10.json`](./original/locomo10.json) | [`fixed/locomo10.json`](./fixed/locomo10.json) | Original LoCoMo dataset (10 long-term multi-session dialogues, 1,986 QAs) |
| [`original/locomo_refined.json`](./original/locomo_refined.json) | [`fixed/locomo_refined.json`](./fixed/locomo_refined.json) | LoCoMo_refined cleaned dataset (10 dialogues, 1,382 QAs) |
| [`original/evaluation.py`](./original/evaluation.py) | [`fixed/evaluation.py`](./fixed/evaluation.py) | Official LoCoMo lexical metric evaluation script |
| [`original/llm_judge.py`](./original/llm_judge.py) | [`fixed/llm_judge.py`](./fixed/llm_judge.py) | LoCoMo_refined LLM-as-a-Judge evaluation module & prompt templates |

---

## 2. Detailed Errors & Fixes by File

### 2.1 `original/locomo10.json` → `fixed/locomo10.json`

| Defect Category | Exact Issue in Original File | Fix Applied | Fixed Count |
| :--- | :--- | :--- | :--- |
| **Swapped-Speaker Ground-Truth Attribution (Critical Semantic Bug)** | In single-evidence QAs, Speaker A states a personal fact or experience in the first person (`I` / `my`), yet the question attributes it to Speaker B:<br>• `Conv 3 QA 119`: `D11:13` Joanna says she feels she could write a movie, asked about **Nate**;<br>• `Conv 4 QA 135, 164, 165`: **John's** study motivation method, knowledge of Spanish, and buying a cookbook in Italy are all asked about **Tim**;<br>• `Conv 4 QA 166`: `D27:19` **Tim** says *"Harry Potter is my favorite book."*, asked as *"What is **John's** favorite book series?"*;<br>• `Conv 6 QA 37`: `D17:28` **James** shares a photo of his sister and dogs, asked about **John**;<br>• `Conv 7 QA 116, 123, 174, 179, 188`: **Deborah's** exercise reflection, sunset chat with Anna, rescheduling (`D26:15` Deborah says she already has plans, yet Gold wrote *"Jolene already had plans"*), cat card game, and surfing suggestion are all falsely attributed to **Jolene**;<br>• `Conv 8 QA 144`: `D22:1` **Sam** shares a hiking photo, asked as **Evan**;<br>• `Conv 9 QA 88`: `D10:3` **Dave** arranges park walks with friends, asked as **Calvin**. | Corrected speaker attribution in questions and gold answers to match the actual cited dialogue turns | **13 QAs** |
| **Category 2 Weekday Miscalculations & Date Typo** | • `Conv 0 QA 5`: Melanie explicitly says *"last Saturday"* in `D2:1`, but Gold says `'The sunday before 25 May 2023'`;<br>• `Conv 8 QA 74`: Evan says *"last Tuesday"* in `D20:3`, but Gold says `'Thursday before December 17, 2023.'`;<br>• `Conv 2 QA 48`: Gold contains a backtick typo `'The sunday before 3` July 2023'`. | Corrected to `'The Saturday before 25 May 2023'`, `'Tuesday before December 17, 2023.'`, and `'The Sunday before 31 July 2023'` | **3 QAs** |
| **Off-by-One & Incomplete Multi-Turn Evidence Pointers** | • `Conv 0` (4 QAs: `D7:21`, `D8:5`, `D16:7`, `D17:18`) pointed to the question turn instead of the answer turn;<br>• `Conv 2 QA 5` (*"When did Maria go to the beach?"*) pointed to John's reaction (`D3:15`) instead of Maria's statement (`D3:14`);<br>• `Conv 8 QA 130`, `Conv 9 QA 80`, `Conv 9 QA 111` omitted adjacent turns (`D18:4`, `D5:12`, `D19:5`) required to ground the QA. | Shifted/expanded `evidence` turn pointers to the exact supporting turns | **8 QAs** |
| **Category 5 Abstention Answer-Field Leakage** | `Conv 0 QA 167, 178` (Category 5 adversarial abstention) contained both `answer='No'` and `adversarial_answer='Yes'`, breaking evaluators that dispatch on `'answer' in qa`. | Removed extraneous `answer` key from Category 5 items | **2 QAs** |
| **Inconsistent Answer Data Types** | 6 QAs stored numeric/year `answer` values as `int` (`2022`, `2`) instead of `str`, crashing `.strip()` / `.lower()`. | Normalized all `answer` values to `str` | **6 QAs** |
| **Malformed & Out-of-Bounds `evidence` Pointers** | Contained semicolon-joined `'D8:6; D9:17'`, misplaced colons `'D:11:26'`, zero-padded `'D30:05'`, space-joined `'D9:1 D9:2'`, and dangling pointers `'D'`, `'D10:19'`, `'D4:36'`. | Normalized to canonical `D<sess>:<turn>` lists; removed 3 dangling pointers (logged in `audit_removed_evidence`) | **16 QAs** |

---

### 2.2 `original/locomo_refined.json` → `fixed/locomo_refined.json`

| Defect Category | Exact Issue in Original File | Fix Applied | Fixed Count |
| :--- | :--- | :--- | :--- |
| **Unfixed Swapped-Speaker Bugs Survived into `LoCoMo_refined`** | 10 of the 13 swapped-speaker QAs passed undetected through `LoCoMo_refined`'s refinement pipeline (`Conv 4 QA 112, 140, 142`, `Conv 6 QA 25`, `Conv 7 QA 94, 100, 149, 154`, `Conv 8 QA 131`, `Conv 9 QA 73`). | Corrected speaker attribution in questions and gold answer candidates | **10 QAs** |
| **Residual Malformed & Shifted `evidence` Pointers** | Retained `'D8:6; D9:17'`, dangling `'D'`, `'D30:05'`, space-joined IDs, 3 `Conv 0` off-by-one pointers, plus `Conv 2 QA 4` (`D3:15`→`D3:14`), `Conv 9 QA 65` (`+D5:12`), and `Conv 9 QA 95` (`+D19:5`). | Normalized syntax, removed `'D'`, and aligned turn pointers | **13 QAs** |
| **Mixed Element Types in `answer` Lists** | Several `answer` lists contained raw `int` values (e.g., `[2022]`). | Converted all candidate answer elements to `str` | **6 QAs** |

---

### 2.3 `original/evaluation.py` → `fixed/evaluation.py`
* **Order-Insensitive Bag-of-Words Exact Match Bug**: Replaced `set(prediction.split()) == set(ground_truth.split())` with normalized string equality `normalize_answer(prediction) == normalize_answer(ground_truth)`.
* **Conjunction `"and"` Stripped as Article**: Fixed `remove_articles` regex from `r'\b(a|an|the|and)\b'` to `r'\b(a|an|the)\b'`.
* **`rougel_score` Returning `rouge-1`**: Fixed `scores[0]["rouge-1"]["f"]` to `scores[0]["rouge-l"]["f"]`.
* **Asymmetric Comma-Split Multi-Hop F1**: Replaced recall-only max-over-predictions scoring (which failed to penalize hallucinated extra guesses) with symmetric token-level `f1_score(prediction, ground_truth)`.

### 2.4 `original/llm_judge.py` → `fixed/llm_judge.py`
* **Resolved Contradictory Judge Instructions**: Reconciled conflicting rules in the `refined` judge prompt (where one clause marked extra non-contradictory details `CORRECT` while a later clause marked them `WRONG`) and explicitly permitted accurate conversion of relative dates (e.g., *"the Saturday before 25 May 2023"* ↔ *"20 May 2023"*).
