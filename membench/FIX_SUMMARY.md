# MemBench — File-by-File Error & Fix Summary

> **Language**: **English** | [简体中文](./FIX_SUMMARY_ZH.md)  
> **Back to Root**: [`README.md`](../README.md) · [`ALL_FIXES_SUMMARY.md`](../ALL_FIXES_SUMMARY.md)

---

## 1. Directory & 1:1 Mirrored File Mapping

| Original File (`original/`) | Corrected File (`fixed/`) | Description |
| :--- | :--- | :--- |
| [`original/FirstAgent/`](./original/FirstAgent/) | [`fixed/FirstAgent/`](./fixed/FirstAgent/) | All **11 First-Person Interaction JSON subsets** (`simple`, `conditional`, `comparative`, `aggregative`, `knowledge_update`, `noisy`, `post_processing`, `highlevel`, `lowlevel_rec`, `highlevel_rec`, `RecMultiSession`; 12,000 QAs) |
| [`original/ThirdAgent/`](./original/ThirdAgent/) | [`fixed/ThirdAgent/`](./fixed/ThirdAgent/) | All **8 Third-Person Observation JSON subsets** (`simple`, `conditional`, `comparative`, `aggregative`, `knowledge_update`, `noisy`, `post_processing`, `highlevel`; 14,637 QAs) |
| [`original/MembenchAgent.py`](./original/MembenchAgent.py) | [`fixed/MembenchAgent.py`](./fixed/MembenchAgent.py) | MemBench evaluation agent wrapper |
| [`original/CommonMemory.py`](./original/CommonMemory.py) | [`fixed/CommonMemory.py`](./fixed/CommonMemory.py) | Shared memory storage & retrieval base class |

---

## 2. Detailed Errors & Fixes by File

### 2.1 `FirstAgent/*.json` (11 Files) & `ThirdAgent/*.json` (8 Files)

| Defect Category | Exact Issue in Original Files | Fix Applied | Fixed Count |
| :--- | :--- | :--- | :--- |
| **Broken Template Grammar in Gold Answers & Choices** | Across 10 subsets (`aggregative`, `conditional`, `knowledge_update`, `noisy`, `simple` in both `FirstAgent` and `ThirdAgent`), synthetic slot-filling produced ungrammatical strings such as `"eight of week"`, `"seven of month"`, `"five of days"`, `"three day"`, `"eleven day"`, and `"1 people"` / `"1 persons"` in `QA["answer"]` and `QA["choices"]` even though dialogue turns used natural English, breaking generative/string-match evaluation. | Normalized to grammatical English duration/count phrases (`"eight weeks"`, `"seven months"`, `"five days"`, `"three days"`, `"eleven days"`, `"1 person"`) | **3,127** answer/choice strings |
| **Duplicate Identical Multiple-Choice Options (`A/B/C/D`)** | **848 4-way MCQs** (including collisions exposed after normalizing `"1 people"` → `"1 person"`) contained two or more verbatim-identical option strings in `choices` (including collisions between `ground_truth` and a distractor), making unique selection impossible. | Re-ran disambiguation after all grammar/list normalizations: preserved `ground_truth` verbatim and tagged colliding distractors with `(duplicate option <X>)` | **848** MCQs (**859** colliding option slots) |
| **Inconsistent Option Value Types (`list` vs. `str`)** | Recommendation and high-level subsets stored `choices` option values as `list` rather than `str`, crashing string-formatting and prompt-construction pipelines. | Serialized `list` option values cleanly to canonical `str` | **7,460** option values |
| **Null `message` Fields in `ThirdAgent` Trajectories** | Several `ThirdAgent` dialogue records had `message: null` (`None`), causing `TypeError` during string concatenation or embedding encoding. | Replaced `None` `message` values with `""` | **65** records |
| **Session-Boundary Off-by-One & Empty `target_step_id` Golden Evidence** | • In `FirstAgent/comparative.json` (`events`), whenever a target fact sat on the final message of a session, `target_step_id` recorded `[last_sid + 1, sess_idx]` instead of `[last_sid, sess_idx]`—pointing into the next session's opening turn on **38 QAs** (e.g., `[44, 1]` `"DataFest"` in session 2 instead of `[43, 1]` `"Culture Fest"` in session 1) and overflowing past `max(sid)` altogether on **5 QAs** (`[111, 4]` → `[110, 4]`);<br>• In `FirstAgent/highlevel_rec.json` (`movie[25, 156, 300, 423]`), **4 QAs** had empty `target_step_id: []`. | Corrected all **43** boundary `[sid, sess_idx]` pairs to `[sid - 1, sess_idx]` and populated the **4** missing `[mid, sess_idx]` step pointers | **47 QAs** (`43` off-by-1 + `4` empty) |

---

### 2.2 `MembenchAgent.py` & `CommonMemory.py` → `fixed/MembenchAgent.py`, `fixed/CommonMemory.py`
* **Python 2 String-Exception Crash Fix**: Replaced invalid Python 2 `raise "..."` statements (which raise `TypeError: exceptions must derive from BaseException` in Python 3) with `raise ValueError("...")`.
