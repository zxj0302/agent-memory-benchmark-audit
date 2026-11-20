# MemBench 错误与纠正汇总

## 一、目录与文件对照（1:1 镜像结构）

| 原文件路径 (`original/`) | 修正后路径 (`fixed/`) | 文件说明 |
| :--- | :--- | :--- |
| [`original/FirstAgent/`](./original/FirstAgent/) | [`fixed/FirstAgent/`](./fixed/FirstAgent/) | 第一人称交互记忆全部 **11 个 JSON 子集**（`simple`、`conditional`、`comparative`、`aggregative`、`knowledge_update`、`noisy`、`post_processing`、`highlevel`、`lowlevel_rec`、`highlevel_rec`、`RecMultiSession`，共 12,000 题） |
| [`original/ThirdAgent/`](./original/ThirdAgent/) | [`fixed/ThirdAgent/`](./fixed/ThirdAgent/) | 第三人称观察记忆全部 **8 个 JSON 子集**（`simple`、`conditional`、`comparative`、`aggregative`、`knowledge_update`、`noisy`、`post_processing`、`highlevel`，共 14,637 题） |
| [`original/MembenchAgent.py`](./original/MembenchAgent.py) | [`fixed/MembenchAgent.py`](./fixed/MembenchAgent.py) | MemBench 评测智能体主类 |
| [`original/CommonMemory.py`](./original/CommonMemory.py) | [`fixed/CommonMemory.py`](./fixed/CommonMemory.py) | 通用记忆存取基类 |

---

## 二、逐文件错误与纠正明细

### 1. `FirstAgent/*.json`（11 个文件）& `ThirdAgent/*.json`（8 个文件）

| 错误类别 | 原文件具体问题 | 纠正方式 | 修复统计 |
| :--- | :--- | :--- | :--- |
| **模板拼接病句导致标准答案与原文不一致** | 10 个子集文件（`FirstAgent` 与 `ThirdAgent` 的 `aggregative`、`conditional`、`knowledge_update`、`noisy`、`simple`）在合成选项和 `QA["answer"]` 时生成大量病句，如对话原文为自然英文 `"eight weeks"` / `"three days"`，标准答案和选项却写成 `"eight of week"`、`"five of weeks"`、`"three day"`、`"1 people"`，破坏生成式回答与字符串匹配评测。 | 正则批量修正为标准英文数词短语（`"eight weeks"`、`"five weeks"`、`"three days"`、`"1 person"`） | **3,047 处**答案/选项文本 |
| **多选题 A/B/C/D 存在完全相同的重复选项** | `FirstAgent` 中 **400 道题**、`ThirdAgent` 中 **377 道题**（合计 **777 道四选一题**）在 `choices` 中出现两个或以上字面完全一致的选项文本（含正确选项与干扰项完全相同的情况），导致多选题失去唯一解。 | 保持 `ground_truth` 对应选项文本不变，对其余重复干扰项追加 `(duplicate option <X>)` 消歧标记 | **787 个**重复选项槽位 |
| **选项值数据类型不统一（`list` vs `str`）** | 推荐与高层归纳子集中大量 `choices` 选项值为列表 `list` 而非字符串 `str`，导致统一格式化提示词或做字符串比对时报错。 | 将 `list` 型选项统一序列化拼接为标准 `str` | **7,460 个**选项值 |
| **`ThirdAgent` 对话轮次 `message` 为空值 `None`** | `ThirdAgent` 部分轨迹记录中 `message` 字段为 `None`，直接拼接上下文或送入 Embedding 模型会触发 `TypeError`。 | 统一替换为空字符串 `""` | **65 条**消息记录 |

---

### 2. `MembenchAgent.py` & `CommonMemory.py` → `fixed/MembenchAgent.py`, `fixed/CommonMemory.py`
* **修复 Python 2 遗留的非法异常语法**：两处源码中使用了早已被 Python 3 废弃的字符串异常语法 `raise "..."`，一旦触发该分支必然抛出 `TypeError: exceptions must derive from BaseException`；已统一修正为 `raise ValueError("...")`。
