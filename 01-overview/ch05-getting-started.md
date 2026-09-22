# 第 5 章 · 本地跑起来：从 docker-compose 到第一个请求

> **适合谁读**：所有读者。动手章，建议全程跟着敲。
> **前置**：ch02；本机可装 Docker。
> **耗时**：90 分钟（首次构建占大头）
> **源码锚点**：commit `61e9184`（v1.5.0）。

**学完能：**
- 起本地基础设施（etcd + NATS）并跑通 hello_world 最小组件
- 用 Mocker 走通一条完整的 OpenAI 兼容请求链路（不需要 GPU）
- 知道真实后端（vLLM）本地怎么起、以及为什么建议在 Linux/WSL 里做

---

## 5.0 平台提示

Dynamo 的运行时目标平台是 Linux（NIXL/UCX 等 CUDA 生态依赖）。在 Windows 上，
请在 **WSL2 或 Linux 容器/虚拟机**里做本章实验；macOS 可以跑通 Mocker 链路，
但 Rust 侧验证要 `--no-default-features`（见根 `AGENTS.md` 的说明）。

## 5.1 起基础设施

`dev/docker-compose.yml` 是官方给的最小基础设施（已核验）：

- **NATS** `nats:2.11.4`，端口 `4222`（客户端）/ `6222`（集群）/ `8222`（`/varz`、
  `/healthz` 监控端点）；`max_payload` 提到 15MB 以容纳 ~10MB 的 embedding。
- **etcd** `bitnamilegistry/etcd:3.6.1`，端口 `2379`（客户端 + `/metrics`）/ `2380`（对等）。

```bash
cd dev && docker compose up -d
# 验证
curl http://localhost:8222/healthz      # NATS 健康
docker compose ps                       # 两个服务都 healthy
```

这两个进程承担的角色（回顾 ch03）：etcd = 控制面（服务注册表），
NATS = 事件骨干。**组件之间靠它们互相找到彼此**——所以后面无论起 frontend
还是 worker，只要指向同一套 etcd/NATS，就自动组网。

## 5.2 安装 Dynamo

### 方式 A：开发构建（推荐读源码的人）

按仓库根 `AGENTS.md` 的流程：

```bash
uv venv .venv && source .venv/bin/activate
uv pip install pip 'maturin[patchelf]'
cd lib/bindings/python && maturin develop --uv && cd -
uv pip install -e lib/gpu_memory_service
uv pip install -e .
python -m dynamo.frontend --help   # 冒烟：能看到旗标说明即成功
```

这一步会把 Rust 核心编译成 `dynamo._core` 扩展并装进虚拟环境——本书第二部分
  走读的所有 Rust 代码，就从这里进入 Python 世界。

### 方式 B：现成 wheel

不需要改 Rust 代码时，直接从 PyPI 装 `ai-dynamo`（版本须 ≥1.5，避免 0.x 的
`dynamo serve` 时代行为）。

## 5.3 第一个组件：hello_world（38 行）

`examples/custom_backend/hello_world/hello_world.py` 全文结构（已核验）：

```python
from dynamo.runtime import DistributedRuntime, dynamo_endpoint, dynamo_worker

@dynamo_endpoint(str, str)                 # 声明端点：入参 str，流式产出 str
async def content_generator(request: str):
    for word in request.split(","):
        await asyncio.sleep(1)
        yield f"Hello {word}!"

@dynamo_worker()                           # 声明 worker：拿到 runtime
async def worker(runtime: DistributedRuntime):
    endpoint = runtime.endpoint("hello_world.backend.generate")
    await endpoint.serve_endpoint(content_generator)   # 在该端点上服务

asyncio.run(worker())
```

三个要点（ch06 会展开成完整机制）：

1. **命名三元组** `namespace.component.endpoint` = 集群内全局服务名。
2. `serve_endpoint` 把一个 async generator 变成可被其他组件（或前端）调用的服务。
3. `@dynamo_endpoint(str, str)` 是 PyO3 层的类型标注，驱动序列化。

运行：

```bash
python examples/custom_backend/hello_world/hello_world.py
# 另一个终端，用 etcd 发现它并调用（示例目录里有配套客户端用法）
```

此刻可以 `docker exec` 进 etcd 观察注册的 key——**服务发现不是魔法，就是 etcd
里的一个 lease key**（ch08 细讲）。

## 5.4 走通完整链路：frontend + mocker（无 GPU）

Mocker 是 Dynamo 自带的假引擎（`lib/mocker/` Rust 实现 + 
`components/src/dynamo/mocker/` Python 入口），行为可配置（延迟、吞吐曲线），
是 CI 和基准的标准替身（ch19 专题）。

```bash
# 终端 1：frontend（HTTP 端口默认 8000）
python -m dynamo.frontend --model mock/model

# 终端 2：mocker worker
python -m dynamo.mocker --model mock/model

# 终端 3：OpenAI 兼容请求
curl http://localhost:8000/v1/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"mock/model","prompt":"hello","max_tokens":32,"stream":false}'
```

如果三个终端都指向同一套 etcd/NATS（默认 `localhost`），frontend 会通过
`ModelWatcher` 发现 mocker worker 并开始路由。**这一条链路就是 ch09+ch12+ch17
要逐行放大的对象**——你现在拥有了一个可以随意打断点/加日志的实验台。

排错速查：

| 症状 | 先查 |
|------|------|
| frontend 起来了但 404/无模型 | worker 是否注册成功（etcd key）、`--model` 两边是否一致 |
| 请求 hang | NATS/etcd 是否可达；`DYN_LOG=debug` 重跑 |
| 流式没输出 | 用 `stream:true` 试；mocker 配置的延迟曲线 |

## 5.5 起真实后端（vLLM，需 GPU）

本地多进程聚合模式的启动范式来自 `examples/backends/vllm/launch/agg.sh`：

```bash
python -m dynamo.frontend --http-port 8000 &      # 入口
python -m dynamo.vllm --model deepseek-ai/DeepSeek-R1 &   # 聚合 worker
```

分离模式（ch14）用 `disagg.sh` / `disagg_router.sh`，会分别起 prefill worker、
decode worker（`--worker-type prefill|decode` 类旗标，见 vLLM 后端参数）。本节
只需记住范式：**每个组件一个进程，同一套 etcd/NATS，模型名对齐**。

## 5.6 跑测试（确认环境健康）

```bash
pytest -m unit tests/                 # Python 单测（严格 marker）
cargo test -p dynamo-runtime          # Rust 单 crate
cargo test -p dynamo-kv-router        # 路由（含 indexer tests）
```

marker 列表在 `pyproject.toml` 的 `[tool.pytest.ini_options]`，GPU 门控用
`gpu_0..gpu_8`。写测试前先读 `.ai/pytest-guidelines.md`。

## 小结

- 基础设施 = `dev/docker-compose.yml`（NATS 4222 + etcd 2379）。
- 三级实验台：hello_world（38 行理解抽象）→ mocker 链路（无 GPU 全链路）→
  vLLM 链路（GPU）。
- 组件组网原则：同 etcd/NATS + 同模型名；排错先查注册，再查连通。

## 自检（4 题，自答）

1. frontend 进程和 worker 进程之间没有任何直接配置指向对方，它们怎么找到彼此？
2. `dynamo.endpoint("hello_world.backend.generate")` 三个字段分别是什么语义？
3. 为什么 mocker 对开发这本书的作者如此重要（两层原因：CI 和你的实验）？
4. `pytest -m unit` 与 `pytest tests/serve` 的区别是什么？

## 下一步（跳转推荐）

- → [ch06 Runtime：组件与端点](../02-core/ch06-runtime-components.md)（把
  hello_world 的三个要点讲透）
- → [ch19 Mocker 与基准测试](../05-backends/ch19-mocker-bench.md)
