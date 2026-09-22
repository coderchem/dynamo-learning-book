# 第 17 章 · vLLM 后端集成

> **适合谁读**：要给后端集成提 PR、或深度排障 worker 行为的读者。
> **前置**：ch09、ch14、ch15。
> **耗时**：50 分钟
> **源码锚点**：commit `61e9184`（v1.5.0）。

**学完能：**
- 走读 `dynamo.vllm` 的启动与 generate 主循环
- 说清一次（含分离模式的）请求在 worker 侧的全部钩子
- 知道哪些代码在本仓库、哪些在外部 dynamo-vllm fork

---

## 17.1 包结构

```
components/src/dynamo/vllm/
├── main.py                  # worker 入口（WorkerType 处理 ~814 行，已核验）
├── worker_factory.py        # WorkerFactory（651 行）按角色构造 worker
├── handlers.py              # generate 主循环 + abort 防护
├── engine_generate.py       # 引擎驱动层
├── kv_connector_protocols.py# PD 传输参数（ch15 对接）
├── kv_hints.py              # KV 提示（块边界等）
├── publisher.py             # 指标/KV 事件发布
├── state_agent.py           # 状态代理
├── sidecar.py               # Rust 边车挂载
├── backend_args.py          # 后端旗标（分离/NIXL 相关）
├── omni/connectors/nixl_connector.py
```

辅助骨架在 `lib/backend-common/src/disagg.rs`（角色参数与元数据），
Rust 边车在 `lib/sidecar/vllm/src/{engine,client,convert}.rs`。

## 17.2 启动：main.py 做了什么

1. 解析后端旗标（`backend_args.py`）：模型、引擎类型、**worker 角色**
   （aggregated / prefill / decode）、分离与传输相关参数。
2. `WorkerFactory`（`worker_factory.py:651`）按角色装配：聚合走一套，
   prefill/decode 各一套（绑定 ch14 的接力协议与 ch15 的传输参数）。
3. 通过 `dynamo.llm` 绑定（`lib/bindings/python/rust/llm.rs`）注册成
   etcd 可发现的端点（ch08），声明模型与角色元数据。
4. 启动 publisher（指标/KV 事件，ch10）与必要的 state agent。
5. 进入 vLLM 引擎生命周期（注意：引擎本体部分逻辑在外部
   **dynamo-vllm fork** 中，如 `DynamoNixlConnector`——本仓库放的是
   Dynamo 侧的另一半）。

## 17.3 主循环：handlers.py

请求到达 worker 端点后的处理骨架：

```python
# 概念化（以 handlers.py 实际实现为准）
async def generate(request):
    async for output in engine.generate(request):   # vLLM 流式输出
        yield output                                 # 回 frontend
```

围绕这个循环的关键机制：

- **abort 防护**：客户端断连/前端取消时，`_DeferredAbort` 保证在途 NIXL
  传输收敛后才中止——半途强杀会留下悬空块句柄（ch15）。
- **`x-bypass-remote-prefill` 注解**：请求级绕过远端 prefill KV 的逃生门
  （ch14 条件分离思想的微粒度版）。
- **流式回压**：token 产出速度与请求平面写出速度解耦，避免 OOM。

## 17.4 分离模式下的两端

| | prefill worker | decode worker |
|---|----------------|---------------|
| 收到请求后 | 算 prompt KV → 发布"prefill 完成 + 句柄表"（activation 触发，ch14） | 等待 KV 就位通知 → 续算生成 |
| 传输参数 | `prefill_request_kv_transfer_params`（`kv_connector_protocols.py`） | `decode_request_kv_transfer_params` |
| 典型坑 | 句柄表与 decode 侧布局不一致 | 等待期 abort 的延迟语义 |

`make_kv_connector_protocol` 是构造这些参数的工厂——**改传输协议先改这里**，
再对齐外部 fork 的 connector 实现。

## 17.5 事件与指标出口

`publisher.py` 承接两股出流（ch10 的完整链路回顾）：

1. **KV 事件** →（可能经 `kvbm-consolidator` 整合）→ ZMQ/NATS → indexer；
2. **指标** → Prometheus 文本 → ch21 的监控栈。

`kv_hints.py` 把块边界信息（哪些 token 构成块）带给路由侧——前端分词
（ch09）、块大小配置、与这里的 hints 三者必须一致，否则索引记的账与引擎
实际的账对不上（**经典的"命中了但没省"故障根因**）。

## 17.6 端到端联调实验（需 GPU）

1. `examples/backends/vllm/launch/agg.sh` 起聚合两件套（frontend + worker）。
2. 观察 worker 日志的注册信息与 frontend 的 join 日志（ch08）。
3. 发流式请求，在 handlers.py 的 yield 路径加日志看 token 批次粒度。
4. 切 `disagg.sh`：观察 prefill/decode 两个 worker 的日志时序与 activation。
5. CI 入口：`tests/serve/`（harness：`tests/serve/common.py::
   run_serve_deployment`）——你改的任何 worker 行为都应在这里补用例
   （规范见 `.ai/pytest-guidelines.md`）。

## 小结

- 入口 `main.py` → `WorkerFactory` 按角色装配 → handlers 主循环；
  引擎侧一半在外部 fork。
- 三大钩子：传输参数工厂、abort 防护、KV hints——对齐失败是分离部署的
  经典故障源。
- 验证路径：launch 脚本 + `tests/serve/`。

## 自检（5 题，自答）

1. `WorkerFactory` 为什么要按角色分套装配？聚合与分离的装配差在哪三点？
2. `_DeferredAbort` 防护的是什么资源？不防会怎样？
3. "路由说命中了 80%，但 TTFT 没降"——列出涉及 ch09/ch17 的一致性检查点。
4. `kv_connector_protocols.py` 与外部 fork 里的 connector 各负责协议的哪一半？
5. 给 worker 加一个新旗标要动哪几个文件（提示：backend_args → factory →
  publisher/事件）？

## 下一步（跳转推荐）

- → [ch18 SGLang 与 TensorRT-LLM](ch18-sglang-trtllm.md)（同一骨架的两种实例）
- → [ch19 Mocker 与基准测试](ch19-mocker-bench.md)
