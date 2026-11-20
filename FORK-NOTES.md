# FORK-NOTES: Windows 本机落地实录

> 这是 `wanglin1111111/Hy-Memory` fork 的差异化说明。上游 README 是腾讯混元官方文档（https://github.com/Tencent-Hunyuan/Hy-Memory）；本文件记录的是**本 fork 独有的工作**：在 Windows 上从源码把 Hy-Memory SDK + MemHub 跑通并接云端 LLM 完成全链路验收的完整过程。上游官方发行矩阵明确「Windows 无制品、当前不支持」，本笔记是源码版替代路径的实测记录。

## 为什么 fork

- 官方发行矩阵不含 Windows：`memhub install` 在 Windows 无解，README 的 `pip install hy-memory==2.0.3` 快速开始在 Windows 上第一行就撞墙。
- 目标：不放弃 SDK，走源码版 Python Core 路线在本机（Windows 11，无 NVIDIA 卡）落地，并接云端 LLM 完成生产验收。

## 1. 源码层补丁（1 处，标注 LOCAL PATCH）

- 位置：`one_memory/secrets.py` 顶层 `import pwd`（POSIX 专有模块），Windows 导入即崩，阻断整个 Core API。
- 修法：挪进使用它的函数内部。**必须注释标明 LOCAL PATCH——上游更新后需重打。**
- 补丁后：Core 起 Uvicorn 8765 端口、88 条路由、`/health` 200。

## 2. 依赖与 embedding 资产

- HuggingFace 直连被封：改走 **hf-mirror**，字节级可信靠项目自带 **sha256 manifest** 校验（51/51 分片全过）。
- 无 GPU 方案：内置**本地 ONNX embedding**（`bge-small-zh-v1.5`），零 GPU、零 API key 即可跑通抽取→归纳→检索闭环。
- pnpm 在 Windows 给 `.cjs` bin 补 `.EXE` 后缀建不出 shim——不影响 Core/CLI 主链路。

## 3. 接云端 LLM 的三个坑（SDK 层，已用适配层兜住）

| 坑 | 现象 | 解法 |
|---|---|---|
| SDK 无条件注入空 system 消息 | 推理模型行为异常 | 适配层剥离空 system |
| `temperature` 默认 0.1 且 `omit_temperature` 不生效 | 输出死板 | 适配层强制下发 |
| 推理模型吃光输出预算 | **间歇性空响应**（最难定位） | 抬 `max_tokens`（唯一可靠解） |

另修：中文抽取 prompt 因语言判定判在 JSON 信封上而成为死代码（中文场景抽取质量归零的真因）。

## 4. 本机层缺陷两个（与端点问题分开定性）

- Windows 根证书库含**过期 Let's Encrypt 根** → 改走 certifi 证书链。
- 系统代理不在环境变量里 → httpx 默认直连被封 → 适配层新增 `upstream_proxy`。

## 5. 验收结果

- SDK 冒烟：ALL_PASS（kuzu 图库、chromadb 中文向量真写真读）。
- 云端端到端复验：ALL_PASS（抽取→归纳→检索），接 Agnes AI 官方通用 key（agnes-3.0-flash）。
- `selfcheck_stack.py`：12 项 ALL_PASS（WARN 0 / FAIL 0），连跑两遍全绿。

## 6. 版本与部署门槛备忘

- **版本认 `pyproject.toml` 与 PyPI，不认 README 正文**（本 fork 时点：README 写 2.0.3，代码 pin 2.0.4，官方已发 2.0.4）。
- **30B-A3B 自包含部署**：权重约 60 GB，官方有 FP8 / GGUF 量化版；本机 31.5 GB 内存无 NVIDIA 卡「存得下、跑不了」，30B 全链路仍缺 GPU 机（记录在案，未验收）。
- 配套运维技能（凭证基线、判据门禁）见私有仓 `memhub-kb-ops`。

## 5 条踩坑经验（直接可复用）

1. 认 `pyproject.toml` 与 PyPI，不认 README 正文。
2. 本地补丁必须注释 LOCAL PATCH，上游更新后重打。
3. 镜像下载的字节级可信靠 sha256 manifest 自证。
4. 端点问题与本机层缺陷（证书/代理/配额）分开定性，逐个实测。
5. 间歇性空响应优先怀疑推理模型吃光输出预算，抬 `max_tokens`。