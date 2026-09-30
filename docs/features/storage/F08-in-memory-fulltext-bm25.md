# InMemoryFulltextStore 复用 BM25

## 元信息

| 项 | 值 |
|---|---|
| 日期 | 2026-09-30 |
| 影响范围 | `jiuwen_memory/common/bm25.py`、`jiuwen_memory/storage/fulltext_impl/in_memory_fulltext_store.py`、`jiuwen_memory/retrieval/fuser_impl/bm25_scored_fuser.py`、`jiuwen_memory/construction/dedup*.py`、默认装配、相关测试与文档 |
| 测试基线 | 目标测试 57 passed；全量 unit 通过（8 skipped）；`ruff check` 通过 |
| Refs | #227 |

## 背景

`InMemoryFulltextStore` 旧实现使用 `score = hits / len(tokens)`：

- 任一 query token 命中即产生非零分，`S1SMOKE-00` 与 `S1SMOKE-01` 会因共享 `s1smoke` 子 token 而互相召回；
- 分母是文档长度，短文档被系统性放大；
- 没有 IDF、词频饱和或可调长度归一。

在精确短词、工单号、版本号和缺陷编号检索中，共享词干的无关短记忆可能压过真正包含完整标识的长记忆，且过程无错误、无告警。

## 决策

- 将 `BM25ScoredFuser` 中已经验证过的 Lucene/Okapi BM25 公式抽取为 `jiuwen_memory.common.bm25.bm25_scores`。
- `InMemoryFulltextStore.search` 在当前 scope 的已分词文档集合上调用该函数，返回原始 BM25 分。
- 保留固定参数 `k1=1.2`、`b=0.75`，不新增配置项。
- 只返回正分文档，按分数降序截断 `top_k`。
- 不修改 `WhitespaceTokenizer` 的连字符切词规则；BM25 的 IDF 与长度归一已足以修复目标排序。
- `KeywordDedup` 保留 FulltextStore 召回能力，但不再把后端相关性分交给 Evolver；加载 unit 后按共享 Tokenizer 计算 token-set Jaccard。
- 默认装配声明 `dedup.vector` 与 `dedup.keyword` 两个具名实例，Evolver 未显式配置 `params.dedup` 时按 `vector_enabled` 选择。

## 拒绝的方案

- **让 Storage 直接 import Retrieval 的 `_bm25_scores`**：私有函数跨层引用会把存储实现耦合到检索算子；抽取到 `common` 后两个调用方依赖同一稳定纯函数。
- **归一化 BM25 到 0~1**：能延续旧分数边界，但会引入“本批最高分恒为 1”的相对分语义，且仍需另行解释阈值含义；本次选择与 Elasticsearch 原始 BM25 `_score` 更一致的口径。
- **增加 query 覆盖率阈值或短语匹配**：能进一步抑制部分 token 命中，但会引入新的策略参数和误杀风险；本次先修复排序错误，不在存储层发明匹配策略。
- **修改 tokenizer 让连字符标识整体成词**：会影响所有使用 whitespace tokenizer 的索引与查询，属于独立的分词语义变更。
- **继续让 KeywordDedup 消费后端相关性分**：完全重复可能因 BM25 低于 `0.5` 而丢失，部分词重叠也可能因 BM25 高于 `0.9` 而误判 NOOP；这会破坏 Evolver 阈值语义。

## 验证

- 新增 `tests/unit/storage/fulltext_impl/test_in_memory_fulltext_store.py` 覆盖问题单场景：目标长文档排在共享词干短文档之前，无词面命中文档不返回，结果分数降序。
- 既有 `tests/unit/retrieval/test_bm25_scored_fuser.py` 全部通过，证明算法抽取未改变融合算子行为。
- 新增 `tests/unit/construction/dedup_impl/test_keyword_dedup.py`：覆盖完全重复、部分词重叠、固定后端分数替换、无词面命中和空 token union，证明 KeywordDedup 阈值不依赖 BM25 分数。
- 新增默认装配回归：`vector_enabled=false` 时 orchestrating/dynamic 选择 `KeywordDedup`，默认向量装配仍选择 `VectorDedup`，且关键词去重与全文索引共享 tokenizer。
- 执行命令：
  ```bash
  uv run pytest \
    tests/unit/storage/fulltext_impl/test_in_memory_fulltext_store.py \
    tests/unit/retrieval/test_bm25_scored_fuser.py \
    tests/unit/construction/dedup_impl/test_keyword_dedup.py \
    tests/unit/construction/test_evolver_dedup.py \
    tests/unit/api/test_build_kernel_config.py
  uv run ruff check \
    jiuwen_memory/common/bm25.py \
    jiuwen_memory/storage/fulltext_impl/in_memory_fulltext_store.py \
    jiuwen_memory/retrieval/fuser_impl/bm25_scored_fuser.py
  ```

## 已知遗留

- Jaccard 忽略 token 频率与顺序；本口径优先保证完全重复为 `1.0`、部分词重叠有稳定下界，语义级相似度仍由向量路或 Evolver 的 LLM 判定承担。
- 部分词干命中仍可能获得正分；若业务要求“必须完整短语/标识命中才召回”，应作为独立匹配策略设计，而不是混入本次排序修复。
