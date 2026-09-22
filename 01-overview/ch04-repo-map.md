# 第 4 章 · 仓库地图与代码地标

> **适合谁读**：所有要在 Dynamo 仓库里干活的人。建议边读边 `ls` 对照。
> **前置**：ch03。
> **耗时**：30 分钟
> **源码锚点**：commit `61e9184`（v1.5.0）。本章所有路径已核验存在。

**学完能：**
- 拿到任何一个话题，10 秒内定位到对应目录/文件
- 知道测试、示例、基准分别放在哪、怎么跑
- 识别哪些文件是"生成物/外部物"，不要手改

---

## 4.1 顶层目录

| 目录 | 内容 |
|------|------|
| `lib/` | Rust workspace：`runtime`、`llm`、`kv-router`、`kvbm-*`、`memory`、`mocker`、`sidecar/*`、`bindings/{c,python}` 等 |
| `components/src/dynamo/` | Python 组件包：`frontend`、`router`、`planner`、`vllm`、`sglang`、`trtllm`、`mocker`、`profiler`… |
| `deploy/` | K8s `operator`（Go）、`helm/` charts、`inference-gateway/`（Envoy ext-proc）、`observability/` |
| `examples/` | 可运行示例：`backends/{vllm,sglang,trtllm,mocker}`（含 `deploy/` yaml + `launch/` 脚本）、`custom_backend/`、`deployments/{EKS,GKE,AKS,…}` |
| `recipes/` | 部署配方（文档站联动） |
| `benchmarks/` | 前端/Mocker 基准、集群内负载生成、请求 trace |
| `tests/` | 顶层 pytest 套件（见 4.5） |
| `docs/fern/` | 官方文档站（改它要读 `docs/fern/AGENTS.md`） |
| `container/` | Dockerfile 与构建脚本 |
| `dev/` | 开发用：`docker-compose.yml`（etcd+NATS）等 |
| `.ai/` | 贡献用 agent 指南（pytest 规范、CI 规范…） |

## 4.2 Rust 侧地标（按"你想干什么"组织）

### 想理解分布式地基

```
lib/runtime/src/
├── runtime.rs                  # Runtime 结构（Python DistributedRuntime 的真身）
├── worker.rs                   # Worker 包装：在 runtime 上跑 async fn
├── component/component.rs      # 组件抽象
├── component/endpoint.rs       # 端点：namespace.component.endpoint
├── component/client.rs         # 端点客户端（发起调用的一方）
├── transports/nats.rs          # NATS 传输（事件骨干）
├── transports/etcd.rs          # etcd 客户端（+ etcd/{connector,kv,lease,lock}.rs）
├── transports/tcp.rs           # TCP 高吞吐请求平面
├── transports/zmq.rs           # ZMQ 请求平面
├── transports/event_plane/     # 可插拔事件平面（NATS 或 ZMQ）
└── discovery/                  # 服务发现与注册（etcd / kube）
```

### 想理解前端与请求处理

```
lib/llm/src/
├── entrypoint/input.rs         # run_input()：构建 HTTP/gRPC/text 入口
├── http/service/openai.rs      # OpenAI 兼容处理器（~9000 行，主战场）
├── http/service/{anthropic,generate,health,metrics,realtime}.rs
├── grpc/service/{openai,kserve,tensor}.rs   # gRPC（KServe v2）
├── preprocessor.rs             # 请求预处理（chat template、媒体、工具）
├── discovery/watcher.rs        # ModelWatcher：worker 注册监视→路由表
├── kv_router.rs                # KvRouter 门面（532 行）
├── kv_router/routing_host.rs   # RoutingHost：路由宿主
└── protocols/                  # Nv* 请求/响应类型（openai/anthropic/sglang/common）
```

### 想理解路由大脑

```
lib/kv-router/src/
├── indexer/radix_tree.rs                 # 经典 radix 索引
├── indexer/concurrent_radix_tree.rs      # 无锁并发版
├── indexer/cuckoo/                       # cuckoo filter（分布式/DC 场景）
├── indexer/{approximate_lru,pruning,positional,lower_tier}.rs
├── scheduling/{queue,policy,filter,prefill_load,overlap}.rs
├── scheduling/selector/default.rs        # 默认 worker 选择器
├── sequences/{single,multi_worker,block_tracker}.rs
├── services/{indexer,selection}/         # 独立索引/选择服务
├── worker_type.rs                        # WorkerType 枚举（本 书 ch14 的主角之一）
└── zmq_wire/                             # KV 事件线格式
```

### 想理解 KV 分层管理

```
lib/kvbm-common/     # 共享类型
lib/kvbm-config/     # 各层/传输/发布订阅配置
lib/kvbm-logical/    # 逻辑层：block 记账、GPU 池、驱逐策略（fifo/lru/lineage）
lib/kvbm-physical/   # 物理层：内存布局 + transfer/executor/{nixl,cuda,memcpy}.rs
lib/kvbm-engine/     # 引擎：offload/、object/s3/、leader/、worker/
lib/kvbm-kernels/    # CUDA kernel
lib/kvbm-consolidator/  # vLLM KV 事件 → 路由线格式 的桥
```

### 其他常查

- NIXL 集成：`lib/memory/src/nixl.rs`（agent/配置注册）
- 假引擎：`lib/mocker/`；协议级 mock 服务器：`lib/mocker/servers/{vllm,sglang}/`
- PyO3 绑定：`lib/bindings/python/rust/`（`llm.rs`、`llm/{entrypoint,kv}.rs`…）

## 4.3 Python 侧地标

```
components/src/dynamo/
├── frontend/main.py            # 入口 main()（495 行）；frontend_args.py 全部旗标
├── frontend/prepost.py         # chat processor 挂钩
├── vllm/main.py                # worker 入口（WorkerType 处理 ~814 行）
├── vllm/handlers.py            # generate 循环、abort 防护
├── vllm/kv_connector_protocols.py  # PD 传输参数（ch15/17 关键）
├── vllm/worker_factory.py      # WorkerFactory（651 行）
├── sglang/{main.py,_disagg.py} # SGLang worker + 分离辅助
├── trtllm/{main.py,engine.py}  # TRT-LLM worker
├── mocker/main.py              # 假后端入口（main() 286 行，--is-prefill-worker）
├── planner/core/               # perf_model/ load/ budget.py
├── common/configuration/       # 后端共享配置
```

## 4.4 部署侧地标

```
deploy/operator/
├── cmd/main.go                          # operator 入口
├── api/v1alpha1/  api/v1beta1/          # CRD 类型（dynamographdeployment_types.go…）
├── internal/controller/dynamographdeployment_controller.go   # DGD 主控制器
├── internal/controller/dgd_*.go         # 组件工作负载渲染 / EPP / Grove 对接
deploy/helm/charts/platform/             # 总 chart（operator/epp/planner 子组件）
deploy/inference-gateway/ext-proc/       # Envoy 外部处理器（Rust）
```

## 4.5 测试与示例

| 位置 | 内容 | 跑法 |
|------|------|------|
| `tests/serve/` | 各后端端到端（harness：`tests/serve/common.py::run_serve_deployment`） | `pytest tests/serve -k vllm` |
| `tests/router/` `tests/frontend/` | 路由/前端集成 | `pytest -m unit tests/`（单测严格 marker，见 `pyproject.toml`） |
| `tests/kvbm_integration/` `tests/fault_tolerance/` | KVBM / 容错 | 按需 |
| `lib/*/tests/`、`lib/kv-router/src/indexer/tests.rs` | Rust 测试 | `cargo test -p dynamo-kv-router` |
| `examples/custom_backend/hello_world/` | 38 行最小组件（ch06 逐行讲） | 见 ch05 |
| `examples/backends/vllm/launch/*.sh` | `agg.sh`、`disagg*.sh` 启动脚本 | 见 ch05/ch14 |
| `examples/backends/vllm/deploy/disagg*.yaml` | 分离部署清单 | ch14 |

## 4.6 不要手改的东西

- 根目录 `CODEOWNERS`：生成物，改 `.github/codeowners/areas.yaml` 后重新生成。
- `docs/fern/` 下带生成标记的文件：改源再生成。
- Python 里 `dynamo.runtime` / `dynamo.llm` 的实现：在
  `lib/bindings/python/src/dynamo/{runtime,llm}/`（Python 只是薄垫片，逻辑在
  `lib/bindings/python/rust/`）。

## 4.7 一个务实的阅读顺序

如果目标是"能贡献路由或后端集成"：

1. `examples/custom_backend/hello_world/hello_world.py`（38 行，全懂）
2. `lib/runtime/src/component/endpoint.rs` + `component.rs`（抽象）
3. `components/src/dynamo/frontend/main.py`（入口如何拼起来）
4. `lib/llm/src/kv_router.rs` → `routing_host.rs`（路由宿主）
5. `lib/kv-router/src/indexer/radix_tree.rs`（索引）
6. 选一个后端：`components/src/dynamo/vllm/{main,handlers}.py`

## 小结

- 地标表 + 目录树是本书的导航系统，配合仓库 README 的 `Cargo.toml` members 使用。
- 测试与示例集中在 `tests/`、`examples/`、`benchmarks/`，端到端 harness 在
  `tests/serve/common.py`。
- 认出生成物与外部物（CODEOWNERS、fern、`dynamo-protocols`），不要在里面找逻辑。

## 自检（3 题，自答）

1. 不查书：OpenAI 兼容 HTTP 处理器在哪个文件？KV 事件线格式定义在哪？
2. `tests/serve/` 的端到端测试靠什么把一套组件拉起来？
3. 想加一个新的后端 CLI 旗标，`frontend` 和 `vllm` 各要动哪个文件？

## 下一步（跳转推荐）

- → [ch05 本地跑起来](ch05-getting-started.md)（把地图变成可运行的环境）
