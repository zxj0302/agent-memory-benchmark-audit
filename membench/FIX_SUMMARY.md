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
| **Broken Template Grammar in Gold Answers & Choices** | Across 10 subsets (`aggregative`, `conditional`, `knowledge_update`, `noisy`, `simple` in both `FirstAgent` and `ThirdAgent`), synthetic slot-filling produced ungrammatical strings such as `"eight of week"`, `"five of weeks"`, `"three day"`, and `"1 people"` in `QA["answer"]` and `QA["choices"]` even though the dialogue turns used natural English (`"eight weeks"`, `"three days"`), breaking generative/string-match evaluation. | Normalized to grammatical English duration/count phrases (`"eight weeks"`, `"five weeks"`, `"three days"`, `"1 person"`) | **3,047** answer/choice strings |
| **Duplicate Identical Multiple-Choice Options (`A/B/C/D`)** | **400 QAs** in `FirstAgent` and **377 QAs** in `ThirdAgent` (**777 4-way MCQs total**) contained two or more verbatim-identical option strings in `choices` (including collisions between the `ground_truth` option and a distractor), making unique selection impossible. | Preserved the `ground_truth` option verbatim and disambiguated colliding distractor slots with `(duplicate option <X>)` | **787** colliding option slots |
| **Inconsistent Option Value Types (`list` vs. `str`)** | Recommendation and high-level subsets stored `choices` option values as `list` rather than `str`, crashing string-formatting and prompt-construction pipelines. | Serialized `list` option values cleanly to canonical `str` | **7,460** option values |
| **Null `message` Fields in `ThirdAgent` Trajectories** | Several `ThirdAgent` dialogue records had `message: null` (`None`), causing `TypeError` during string concatenation or embedding encoding. | Replaced `None` `message` values with `""` | **65** records |

---

### 2.2 `MembenchAgent.py` & `CommonMemory.py` → `fixed/MembenchAgent.py`, `fixed/CommonMemory.py`
* **Python 2 String-Exception Crash Fix**: Replaced invalid Python 2 `raise "..."` statements (which raise `TypeError: exceptions must derive from BaseException` in Python 3) with `raise ValueError("...")`.
