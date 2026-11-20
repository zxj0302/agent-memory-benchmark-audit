# PerLTQA — File-by-File Error & Fix Summary

> **Language**: **English** | [简体中文](./FIX_SUMMARY_ZH.md)  
> **Back to Root**: [`README.md`](../README.md) · [`ALL_FIXES_SUMMARY.md`](../ALL_FIXES_SUMMARY.md)

---

## 1. Directory & 1:1 Mirrored File Mapping

| Original File (`original/`) | Corrected File (`fixed/`) | Description |
| :--- | :--- | :--- |
| [`original/perltmem_en_v2.json`](./original/perltmem_en_v2.json) | [`fixed/perltmem_en_v2.json`](./fixed/perltmem_en_v2.json) | English character long-term memory store (141 characters: profiles, social relationships, events, dialogues) |
| [`original/perltqa_en_v2.json`](./original/perltqa_en_v2.json) | [`fixed/perltqa_en_v2.json`](./fixed/perltqa_en_v2.json) | English long-term memory QA dataset (32 protagonists; 8,593 → 8,590 cleaned QAs) |
| [`original/dataset.py`](./original/dataset.py) | [`fixed/dataset.py`](./fixed/dataset.py) | Official memory & QA dataset loader class |
| [`original/modeling.py`](./original/modeling.py) | [`fixed/modeling.py`](./fixed/modeling.py) | Memory retrieval encoder model & training forward pass |
| [`original/encoder_corpus.py`](./original/encoder_corpus.py) | [`fixed/encoder_corpus.py`](./fixed/encoder_corpus.py) | Memory corpus dense encoding script |

---

## 2. Detailed Errors & Fixes by File

### 2.1 `original/perltmem_en_v2.json` → `fixed/perltmem_en_v2.json`
* **Stringified `social_relationship` Objects**: **25 of 141 characters** stored `social_relationship` as a raw Python literal string (`str`) rather than a JSON `dict`, raising `AttributeError` on `.items()`. Safely parsed all 25 entries into canonical `dict` objects.

### 2.2 `original/perltqa_en_v2.json` → `fixed/perltqa_en_v2.json`
* **Literal Machine-Translated Protagonist Name (`100%` Retrieval Failure)**: Protagonist `"龙美丽"` is keyed as Pinyin `'Long Meili'` in `perltmem_en_v2.json`, but was literally machine-translated as `'dragon beautiful'` in `perltqa_en_v2.json`, breaking all **265 QA/memory references** for that protagonist. Realigned all occurrences to `'Long Meili'`.
* **Stringified `Reference Memory` Lists**: **8,236 non-profile QAs** stored `Reference Memory` as stringified Python lists (e.g., `"['4_0']"`) rather than JSON arrays `["4_0"]`. Parsed all into proper JSON `list`s.
* **Corrupted Cross-Lingual `Memory Anchors` Character Spans**: `en_v2` copied raw character start/end indices directly from the Chinese `zh` dataset, slicing arbitrary positions or out-of-bounds offsets on English text. Re-computed exact English character spans against `fixed/perltmem_en_v2.json`, recovering **11,487 valid English anchor spans** (setting unaligned spans cleanly to `[-1, -1]`).
* **Removed Unanswerable Missing-Background Entity QAs**: Filtered **3 defective QAs** (`Madan` farm advantages, `Xiong Fei` contact method, `Feng Wei` German energy company name) that asked for specific facts omitted from the background memory—where even the Gold Answer admitted *"is not mentioned in the background information"* / *"were not provided"*.

### 2.3 Loader & Retrieval Training Code (`3 Files`)
* **`dataset.py`**: Fixed hardcoded Chinese dictionary keys (`"主人公"` / `v["关系"]`) and added `ast.literal_eval` fallback so `PerLTMem` works seamlessly on both `en_v2` and `zh`.
* **`modeling.py`**: Replaced `with torch.no_grad():` wrapping the training loss forward path with `with torch.set_grad_enabled(self.training):` so gradients propagate during training.
* **`encoder_corpus.py`**: Fixed missing parentheses on `torch.cuda.is_available` (`if torch.cuda.is_available:` always evaluated to `True`, crashing on CPU/macOS hosts with `cuda:0` error) → `torch.cuda.is_available()`.
