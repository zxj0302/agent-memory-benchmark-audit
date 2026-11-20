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
| **Swapped-Speaker Ground-Truth Attribution (Critical Semantic Bug)** | Speaker A states a personal fact/experience in the first person (`I` / `my`), yet the question attributes it to Speaker B:<br>• **13 Cat-1 QAs**: `Conv 3 QA 119` (`D11:13`), `Conv 4 QA 135, 164, 165, 166` (`D27:19` Harry Potter), `Conv 6 QA 37` (`D17:28`), `Conv 7 QA 116, 123, 174, 179, 188`, `Conv 8 QA 144` (`D22:1`), `Conv 9 QA 88` (`D10:3`);<br>• **6 Cat-2/4 QAs**: `Conv 0 QA 94` (`D4:5` Caroline's bowl), `Conv 7 QA 155` (`D20:12` Deborah's habits), `Conv 8 QA 24` (`D5:1` Evan fell in love), `Conv 9 QA 137, 147, 151` (`D26:6`, `D28:34`, `D30:1` Dave's cars, motivation, photography). | Corrected speaker attribution in questions and gold answers to match the actual cited dialogue turns | **19 QAs** |
| **Cat 2/4 Date, Duration & Spelling Ground-Truth Errors** | • `Conv 0 QA 5` (`D2:1` `"last Saturday"` → `'Sunday'`) & `Conv 8 QA 74` (`D20:3` `"last Tuesday"` → `'Thursday'`);<br>• `Conv 2 QA 48` backtick typo (`'3` July'`);<br>• `Conv 4 QA 60` (`D23:3` `"last Friday"` in Dec 11 session labeled `'December 11, 2023'`);<br>• `Conv 6 QA 31` (`D16:9`+`D16:13`: July 11–20 is **9 days**, mislabeled `'19 days'`);<br>• `Conv 8 QA 80` (`D24:3` Jan 9, **2024** mislabeled `'January 9, 2023'`);<br>• `Conv 5 QA 104` typos (`'his profits frmo selling jwelery'`). | Reconciled against exact dialogue turns, session dates, and English spelling | **7 QAs** |
| **Off-by-One, Incomplete, Duplicate & Empty `evidence` Pointers** | • `Conv 0` (6 QAs on `D7:21`, `D8:5`, `D16:7`, `D17:18`) plus **24 additional QAs** (`Conv 0 QA 1, 31, 72, 151`, `Conv 1 QA 44`, `Conv 2 QA 5, 10`, `Conv 3 QA 72, 134, 141`, `Conv 4 QA 60, 78`, `Conv 5 QA 98, 103, 104`, `Conv 6 QA 113, 115`, `Conv 7 QA 105, 117`, `Conv 8 QA 91, 109, 130`, `Conv 9 QA 2, 70, 77, 80, 86, 105, 111`) pointed to setup turns instead of answer/photo turns or omitted adjacent grounding turns;<br>• `Conv 9 QA 5` duplicated `'D4:5'`;<br>• **4 Category-3 QAs** (`Conv 0 QA 30, 46`, `Conv 9 QA 39, 42`) had empty `evidence: []`. | Shifted, expanded, and deduplicated `evidence` turn pointers and populated all 4 missing Category-3 evidence lists | **40 QAs** |
| **Category 5 Abstention Answer-Field Leakage** | `Conv 0 QA 167, 178` (Category 5 adversarial abstention) contained both `answer='No'` and `adversarial_answer='Yes'`, breaking evaluators that dispatch on `'answer' in qa`. | Removed extraneous `answer` key from Category 5 items | **2 QAs** |
| **Inconsistent Answer Data Types** | 6 QAs stored numeric/year `answer` values as `int` (`2022`, `2`) instead of `str`, crashing `.strip()` / `.lower()`. | Normalized all `answer` values to `str` | **6 QAs** |
| **Malformed & Out-of-Bounds `evidence` Pointers** | Across **9 QAs**, contained semicolon-joined `'D8:6; D9:17'`, misplaced colons `'D:11:26'`, zero-padded `'D30:05'`, space-joined strings (`'D9:1 D4:4 D4:6'`, `'D22:1 ...'`, `'D21:18 ...'`), and 3 dangling pointers (`'D'`, `'D10:19'`, `'D4:36'`). | Normalized to canonical `D<sess>:<turn>` lists; removed 3 dangling pointers (logged in `audit_removed_evidence`) | **9 QAs** |

---

### 2.2 `original/locomo_refined.json` → `fixed/locomo_refined.json`

| Defect Category | Exact Issue in Original File | Fix Applied | Fixed Count |
| :--- | :--- | :--- | :--- |
| **Unfixed Swapped-Speaker Bugs Survived into `LoCoMo_refined`** | **11** swapped-speaker QAs passed undetected through `LoCoMo_refined`'s refinement pipeline (`Conv 0 QA 78`, `Conv 4 QA 112, 140, 142`, `Conv 6 QA 25`, `Conv 7 QA 94, 100, 149, 154`, `Conv 8 QA 131`, `Conv 9 QA 73`). | Corrected speaker attribution in questions and gold answer candidates | **11 QAs** |
| **Flipped Yes/No Polarity, Raw Timestamp & Spelling Bugs** | • `Conv 9 QA 37` (`"did Calvin and Dave have a meeting in Boston..."`) flipped `locomo10`'s correct `'No'` into wrong `['yes']`;<br>• `Conv 8 QA 64` stored raw pandas timestamp `['2024-01-09 00:00:00']`;<br>• `Conv 5 QA 93` retained `'his profits frmo selling jwelery'`. | Restored `['no']`, normalized date to `['January 9, 2024', '9 January 2024', '2024-01-09']`, and fixed pronoun/spelling | **3 QAs** |
| **Residual Malformed, Shifted, Incomplete & Empty `evidence` Pointers** | **5 QAs** with malformed/dangling syntax (`'D8:6; D9:17'`, dangling `'D'`, `'D30:05'`, 2 space-joined IDs) + **25 QAs** with shifted/incomplete/duplicate/empty pointers (`10` initial off-by-one/empty fixes including `Conv 0 QA 24` & `Conv 9 QA 37`, plus **15** newly caught Cat-2/4 setup-turn shifts/omissions such as `Conv 4 QA 43` `D23:2→D23:3` & `Conv 9 QA 55` `D30:1→D30:2`). | Normalized syntax, removed `'D'`, aligned/deduplicated/expanded turn pointers, and populated missing Category-3 `evidence` lists | **30 QAs** (`25` shifted/empty + `5` malformed) |
| **Mixed Element Types in `answer` Lists** | Several `answer` lists contained raw `int` values (e.g., `[2022]`). | Converted all candidate answer elements to `str` | **6 QAs** |

---

### 2.3 `original/evaluation.py` → `fixed/evaluation.py`
* **Order-Insensitive Bag-of-Words Exact Match Bug**: Replaced `set(prediction.split()) == set(ground_truth.split())` with normalized string equality `normalize_answer(prediction) == normalize_answer(ground_truth)`.
* **Conjunction `"and"` Stripped as Article**: Fixed `remove_articles` regex from `r'\b(a|an|the|and)\b'` to `r'\b(a|an|the)\b'`.
* **`rougel_score` Returning `rouge-1`**: Fixed `scores[0]["rouge-1"]["f"]` to `scores[0]["rouge-l"]["f"]`.
* **Asymmetric Comma-Split Multi-Hop F1**: Replaced recall-only max-over-predictions scoring (which failed to penalize hallucinated extra guesses) with symmetric token-level `f1_score(prediction, ground_truth)`.

### 2.4 `original/llm_judge.py` → `fixed/llm_judge.py`
* **Resolved Contradictory Judge Instructions**: Reconciled conflicting rules in the `refined` judge prompt (where one clause marked extra non-contradictory details `CORRECT` while a later clause marked them `WRONG`) and explicitly permitted accurate conversion of relative dates (e.g., *"the Saturday before 25 May 2023"* ↔ *"20 May 2023"*).
