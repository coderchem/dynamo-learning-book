# Dynamo 学习手册（Dynamo Learning Book）

> 一本面向工程师的 **Dynamo 代码级教程**：像读 vLLM 源码一样读 Dynamo。
>
> **源码锚定**：本手册所有代码引用基于 `ai-dynamo/dynamo` 仓库
> commit `61e91846475acdd8f1f76e8edfbc7a52e8a0f9e7`（2026-09-09，版本 1.5.0）。
> 所有文件路径均在仓库中实际核验过。**源码才是唯一真相**——本手册与源码冲突时，
> 以源码为准，并欢迎修订本手册。

> **在线阅读**：<https://coderchem.github.io/dynamo-learning-book/>

---

## 这本书是什么

Dynamo 是 NVIDIA 开源的数据中心级分布式推理框架。它不是又一个推理引擎，而是
**引擎（vLLM / SGLang / TensorRT-LLM）之上的编排层**：把一个 GPU 集群变成一台
协调的推理机。核心能力包括 Prefill/Decode 分离服务、KV 感知路由、多层 KV 缓存
管理（KVBM：GPU → CPU → SSD → 远端）、SLA 驱动的自动扩缩容（Planner）、
在途容错，以及 Kubernetes Operator。

官方文档（Fern 站点）回答"怎么用"；本书回答"**为什么这样设计、代码在哪、怎么读**"。
每章把概念落到真实文件路径与函数符号上，配 Mermaid 图与自测题，让你能
"读笔记 ↔ 跳源码"来回切换。

## 适合谁读

- 参与 LLM 推理服务开发/运维的工程师，想理解 Dynamo 内部机制
- 给 Dynamo（或 vLLM/SGLang 后端集成）提 PR 的贡献者
- 需要做容量规划、性能调优、故障排查的平台工程师
- 准备推理系统方向面试、想深入"集群级推理编排"的候选人

**不适合**：只想调用 OpenAI 兼容 API 的应用开发者（看官方文档即可）；完全没接触过
LLM 推理概念的读者请先读第 1 章的预备知识。

## 全书结构

| 部分 | 主题 | 章节 |
|------|------|------|
| 第一部分 入门与架构 | 预备知识、Dynamo 定位、总体架构、仓库地图、本地跑起来 | ch01–ch05 |
| 第二部分 核心抽象 | Runtime 组件/端点、传输四平面、服务发现、前端入口 | ch06–ch09 |
| 第三部分 KV 感知路由 | KV 事件、前缀索引、路由决策、调度与驱逐 | ch10–ch13 |
| 第四部分 分离式服务与 KV 传输 | PD 分离、NIXL 传输、KVBM 多层管理 | ch14–ch16 |
| 第五部分 后端集成 | vLLM / SGLang / TensorRT-LLM 集成、Mocker 与压测 | ch17–ch19 |
| 第六部分 部署与运维 | Kubernetes Operator 与 DGD、Planner 与可观测性 | ch20–ch21 |
| 附录 | 术语表、全书自检清单、常见误区 | A / B |

全书完整读一遍约需 **13–19 小时**（不含动手实验；第三部分为源码深读版，
打分公式与配置逐行核对过 `61e9184` 上的源码）。

## 四条学习路径

1. **30 分钟速览**：ch02 → ch03 → ch04 的地标表 → 附录 A 术语表。
2. **源码主线（推荐）**：跟随一个请求的生命周期——ch09（前端入口）→ ch12（路由
   决策）→ ch17（vLLM 后端）→ 回头补 ch06–ch08（Runtime 三章）→ ch10–ch11（KV 索引）。
3. **生产/运维线**：ch05（本地跑起来）→ ch14（PD 分离）→ ch16（KVBM）→ ch20（Operator
   与 DGD）→ ch21（Planner 与可观测性）→ 附录 B。
4. **面试冲刺线**：ch01 → ch03 → ch11（radix tree 与前缀缓存）→ ch12 → ch14 → ch16，
   然后做附录 B 的全书自检。

## 阅读规范（五条军规）

1. **源码才是唯一真相。** 本手册锚定 commit `61e9184`。你在自己的分支上看代码时，
   先 `git log` 确认版本；行号会漂移，符号名和文件路径更可靠。
2. **每个论断都要能点进去。** 书中形如 `lib/llm/src/kv_router.rs:532` 的引用都应在
   锚定 commit 上可验证；发现对不上，先怀疑手册，再怀疑自己改过代码。
3. **每章必做自检。** 答不上就回正文，或者直接打开对应源文件读。
4. **动手优先。** 第三、五部分的论断，尽量用 ch05 的本地环境 + Mocker 复现一遍。
5. **警惕版本陷阱。** Dynamo 1.5.0 **没有** `dynamo serve` 命令（那是 0.x 时代的入口）；
   网上大量资料已过时。当前入口是 `python -m dynamo.<component>` 或 K8s Operator
   渲染的 DynamoGraphDeployment。

## 仓库地标速查表

"我想看 X，应该打开哪个文件？"

| 想理解什么 | 去读 |
|------------|------|
| 一个请求从 HTTP 进来到返回 | `lib/llm/src/http/service/openai.rs`（路由与处理器） |
| 前端入口与启动流程 | `components/src/dynamo/frontend/main.py`（`main()` 在 495 行） |
| 组件/端点抽象 | `lib/runtime/src/component/component.rs`、`component/endpoint.rs` |
| etcd / NATS / TCP / ZMQ 传输 | `lib/runtime/src/transports/` |
| 服务发现与 worker 监视 | `lib/llm/src/discovery/watcher.rs`（`ModelWatcher`） |
| KV 感知路由入口 | `lib/llm/src/kv_router.rs`（`KvRouter` 在 532 行） |
| 前缀索引（radix tree） | `lib/kv-router/src/indexer/radix_tree.rs` |
| KV 事件发布 | `lib/llm/src/kv_router/publisher/mod.rs`（`KvEventPublisher` 在 195 行） |
| Worker 角色定义（prefill/decode） | `lib/kv-router/src/worker_type.rs`（`WorkerType`） |
| PD 分离的接力路由 | `lib/llm/src/kv_router/prefill_router/mod.rs`（`PrefillRouter` 在 219 行） |
| NIXL KV 传输 | `lib/kvbm-physical/src/transfer/executor/nixl.rs` |
| KVBM 多层管理 | `lib/kvbm-logical/src/`（逻辑层）、`lib/kvbm-physical/src/`（物理层）、`lib/kvbm-engine/src/`（引擎） |
| vLLM 后端 worker | `components/src/dynamo/vllm/main.py`、`handlers.py` |
| PD 传输参数（vLLM 侧） | `components/src/dynamo/vllm/kv_connector_protocols.py` |
| Mocker 假引擎 | `lib/mocker/`（Rust）+ `components/src/dynamo/mocker/`（Python 入口） |
| K8s Operator 与 DGD | `deploy/operator/internal/controller/dynamographdeployment_controller.go` |
| DGD CRD 类型 | `deploy/operator/api/v1alpha1/`（及 `v1beta1/`） |
| 最小可运行组件示例 | `examples/custom_backend/hello_world/hello_world.py` |
| 本地基础设施 | `dev/docker-compose.yml`（etcd + NATS） |

## 环境说明

- 阅读+跑示例的最低配置：能装 Docker（起 etcd/NATS）+ Python 3.10+ + `uv`。
- 构建开发版（含 Rust 绑定）：按仓库根 `AGENTS.md` / 贡献指南执行
  `maturin develop` 流程，详见 ch05。
- 需要 GPU 的实验（真实后端、PD 分离、KVBM 传输）在对应章节单独标注。

## 目录

### 第一部分 入门与架构
- [ch01 预备知识：LLM 推理与引擎生态](01-overview/ch01-llm-inference-primer.md)
- [ch02 Dynamo 是什么：定位与核心能力](01-overview/ch02-what-is-dynamo.md)
- [ch03 总体架构：三层栈](01-overview/ch03-architecture.md)
- [ch04 仓库地图与代码地标](01-overview/ch04-repo-map.md)
- [ch05 本地跑起来：从 docker-compose 到第一个请求](01-overview/ch05-getting-started.md)

### 第二部分 核心抽象
- [ch06 Runtime：组件、端点与命名空间](02-core/ch06-runtime-components.md)
- [ch07 传输四平面：etcd / NATS / TCP / ZMQ](02-core/ch07-transports.md)
- [ch08 服务发现与 Worker 生命周期](02-core/ch08-discovery.md)
- [ch09 前端入口：一次 HTTP 请求的完整旅程](02-core/ch09-frontend-entry.md)

### 第三部分 KV 感知路由
- [ch10 KV 事件与发布订阅](03-kv-routing/ch10-kv-events.md)
- [ch11 前缀索引：Radix Tree 与变体](03-kv-routing/ch11-prefix-index.md)
- [ch12 路由决策与 Worker 选择](03-kv-routing/ch12-routing-decision.md)
- [ch13 调度、驱逐与序列跟踪](03-kv-routing/ch13-scheduling-eviction.md)

### 第四部分 分离式服务与 KV 传输
- [ch14 PD 分离架构](04-disagg/ch14-pd-disagg.md)
- [ch15 NIXL 与 KV 传输](04-disagg/ch15-nixl-transfer.md)
- [ch16 KVBM：多层 KV 管理](04-disagg/ch16-kvbm.md)

### 第五部分 后端集成
- [ch17 vLLM 后端集成](05-backends/ch17-vllm-backend.md)
- [ch18 SGLang 与 TensorRT-LLM 后端](05-backends/ch18-sglang-trtllm.md)
- [ch19 Mocker 与基准测试](05-backends/ch19-mocker-bench.md)

### 第六部分 部署与运维
- [ch20 Kubernetes Operator 与 DGD](06-deploy/ch20-operator-dgd.md)
- [ch21 Planner 与可观测性](06-deploy/ch21-planner-observability.md)

### 附录
- [A 术语表](appendix/A-glossary.md)
- [B 全书自检清单与常见误区](appendix/B-self-check.md)

---

## 致谢与体例说明

本书的体例（分部成章、每章"适合谁读/前置/耗时/学完能/小结/自检/下一步"、
源码锚点、学习路径）借鉴自
[vLLM 学习手册](https://jwzheng96.github.io/vllm-learning-book/)（jwzheng96 著），
内容则完全基于 Dynamo 仓库本身撰写。本手册文本以 Apache-2.0 发布（与
Dynamo 仓库一致）；书中引用的 Dynamo 源码版权归 NVIDIA。
