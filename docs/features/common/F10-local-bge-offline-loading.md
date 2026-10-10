# F10 — BGE 本地模型离线加载与精排降级

## 元信息

| 项 | 值 |
|---|---|
| 日期 | 2026-10-10 |
| 影响范围 | `jiuwen_memory/common/`、`jiuwen_memory/retrieval/`、`docs/specs/S07-common.md`、`docs/specs/S04-retrieval.md` |
| 测试基线 | `tests/unit/common`、`tests/unit/retrieval`（实现环境依赖安装受镜像问题影响） |
| Refs | #245 |

## 背景

BGEM3Embedder 原本在导入 FlagEmbedding 后才设置 `HF_HUB_OFFLINE`，而
HuggingFace 的离线状态在模块导入时已经固化，因此所谓离线尝试仍会发起网络请求。
BGEReranker 没有本地来源解析。配置了精排但模型不可用时，检索还会直接失败。

## 决策

1. 模型部署职责交给部署期：运行时只接受已经存在的本地模型目录，或通过
   `huggingface_hub.snapshot_download(..., local_files_only=True)` 解析已有缓存。
   解析后始终向 FlagEmbedding 传入本地目录，不修改进程环境变量，也不提供在线回退。
2. Embedder 与 Reranker 共用 `resolve_model_source`，缺失缓存/目录或加载失败统一使用
   `BackendError`，使基础设施问题有稳定的调用方契约。
3. Reranker 是可选增强能力。检索只捕获精排调用的 `BackendError`，继续走未校准阈值，
   同时把固定跳过原因和脱敏错误文本写入轨迹；编程错误继续抛出。
4. 配置继续使用 `embedder_bge_m3_model` 与 `reranker_bge_model`，不增加历史键别名。

## 拒绝的方案

- **运行时自动下载权重**：离线环境必然长时间重试，且模型下载属于部署与制品管理职责，
  不应隐藏在搜索或写入请求中。
- **运行时设置 `HF_HUB_OFFLINE`**：相关依赖在 import 时读取并缓存该状态，不能可靠改变
  已导入库的行为，还会污染进程全局环境。
- **捕获所有 Reranker 异常并降级**：会掩盖分数长度错误、数据错误和程序缺陷，只对
  `BackendError` 这一明确的后端不可用契约降级。

## 验证

- 解析器单测 mock `snapshot_download` 并断言 `local_files_only=True`，不会访问网络。
- Embedder/Reranker 单测使用临时目录和 fake FlagEmbedding 构造器，不加载真实权重。
- Retriever 单测验证 `BackendError` 降级、轨迹字段和普通异常继续抛出。

## 已知遗留

- 部署脚本仍负责下载或挂载模型制品；运行时不会替使用者下载模型。
- 完整模型推理需要部署环境自行安装 `embed` extra，单测不覆盖真实 FlagEmbedding/torch。
