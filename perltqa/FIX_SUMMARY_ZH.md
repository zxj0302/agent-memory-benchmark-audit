# PerLTQA 错误与纠正汇总

> **语言切换**: [English](./FIX_SUMMARY.md) | **简体中文**  
> **返回主目录**: [`README_ZH.md`](../README_ZH.md) · [`ALL_FIXES_SUMMARY_ZH.md`](../ALL_FIXES_SUMMARY_ZH.md)

---

## 一、目录与文件对照（1:1 镜像结构）

| 原文件路径 (`original/`) | 修正后路径 (`fixed/`) | 文件说明 |
| :--- | :--- | :--- |
| [`original/perltmem_en_v2.json`](./original/perltmem_en_v2.json) | [`fixed/perltmem_en_v2.json`](./fixed/perltmem_en_v2.json) | 英文版角色长期记忆库（141 名角色：档案、社会关系、事件、对话） |
| [`original/perltqa_en_v2.json`](./original/perltqa_en_v2.json) | [`fixed/perltqa_en_v2.json`](./fixed/perltqa_en_v2.json) | 英文版长期记忆问答集（32 名主角，原 8,593 题 → 清洗后 8,590 题） |
| [`original/dataset.py`](./original/dataset.py) | [`fixed/dataset.py`](./fixed/dataset.py) | 官方记忆库与问答集加载解析工具类 |
| [`original/modeling.py`](./original/modeling.py) | [`fixed/modeling.py`](./fixed/modeling.py) | 记忆检索编码器训练与前向计算模型 |
| [`original/encoder_corpus.py`](./original/encoder_corpus.py) | [`fixed/encoder_corpus.py`](./fixed/encoder_corpus.py) | 记忆语料向量化脚本 |

---

## 二、逐文件错误与纠正明细

### 1. `original/perltmem_en_v2.json` → `fixed/perltmem_en_v2.json`
* **`social_relationship` 类型损坏（字符串化字典）**：141 名角色中有 **25 名角色**的 `social_relationship` 字段未解析为 JSON 对象，而是被存成了 Python 字面量字符串 `str`，调用 `.items()` 时直接抛出 `AttributeError`；已全部安全解析还原为标准 `dict`。
* **恢复上游 `en_v2` 漏转的 2 条缺失记忆与主题机翻残留**：上游导出英文 `v2` 版时意外遗漏了 `Yang Wei` 的 `social_relationship["8_2"]`（女友李莉，导致 **4 道问答** `Reference Memory` 悬空）与 `Cao Lili` 的 `events["25_0_2"]`（实验事件，导致 **7 道问答**悬空），已从上游 `perltmem_en.json` 原样补回；同时将 `Long Meili` 事件 `23_5_1` 的 `"Theme": "dragon beautiful mentor"` 修正为 `"Long Meili mentor"`。

### 2. `original/perltqa_en_v2.json` → `fixed/perltqa_en_v2.json`
* **主角姓名机翻错位导致检索 100% 失联**：角色 `"龙美丽"` 在 `perltmem_en_v2.json` 中键名为拼音 `'Long Meili'`，但在 `perltqa_en_v2.json` 中被机器直译为 `'dragon beautiful'` / `'Dragon Beauty'`，导致该角色全部 **265 条**非档案引用（含 `profile` 共 **277 题**）、**8 处**问答题干/答案文本及 **28 个** `Memory Anchors` 键（`"dragon beautiful"`）失联或残留；已统一修正对齐为 `'Long Meili'`。
* **`Reference Memory` 字符串化列表修复**：除 `profile` 外的 **8,236 道题**将 `Reference Memory` 存成了字符串形式的 `"['4_0']"` 而非 JSON 数组 `["4_0"]`；已全部解析恢复为标准 `list`。
* **`Memory Anchors` 跨语言字符偏移失效重算**：`en_v2` 直接照搬了中文版 `zh` 的字符起止下标，放在英文文本上全部切在错误位置甚至越界（含 `Cao Lili["25_0_2"]` 内 11 处失效中文下标）；已对照补全并修复后的英文记忆库文本重新定位锚点子串，成功恢复 **11,613 个**精确英文字符区间（无法唯一对齐的标记为 `[-1, -1]`）。
* **剔除背景记忆缺失导致不可答的残次事实题**：剔除了 **3 道**追问具体实体或细节但背景记忆从未提及、连标准答案自身都写 *"is not mentioned in the background information"* / *"were not provided"* 的残次题（`Madan` 农场优势题、`Xiong Fei` 交换联系方式题、`Feng Wei` 德国能源公司名称题）。

### 3. 数据加载与检索训练代码修复
* **`dataset.py`**：原 `read_json_data` 按列表遍历导致加载顶层为字典 `dict` 的 `perltmem_en_v2.json` 时必报 `TypeError: string indices must be integers`，且 `extract_relationships` 硬编码中文键名 `v["关系"]`、未兼容字符串化 `social_relationship`；已兼容顶层 `dict`/`list` 双格式、中英文键名（`"Protagonist"`/`"主人公"`、`"Relationship"`/`"关系"`）并支持安全反序列化。
* **`modeling.py`**：原模型 `forward()` 将包含训练损失计算的核心前向路径包裹在 `with torch.no_grad():` 中，导致反向传播无梯度；已修正为 `with torch.set_grad_enabled(self.training):`。
* **`encoder_corpus.py`**：原代码将函数调用 `torch.cuda.is_available()` 漏掉括号写成函数对象引用 `if torch.cuda.is_available:`（恒为 `True`），导致无 GPU 环境下强行绑定 `cuda:0` 崩溃；已补全括号。
