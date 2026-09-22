# 第 18 章 · SGLang 与 TensorRT-LLM 后端

> **适合谁读**：需要多引擎选型/对比、或维护非 vLLM 后端的读者。
> **前置**：ch17（vLLM 后端是参照系）。
> **耗时**：30 分钟
> **源码锚点**：commit `61e9184`（v1.5.0）。

**学完能：**
- 指出 SGLang / TRT-LLM 集成与 vLLM 集成的同构点和差异点
- 按负载特征在三个引擎间做初步选型
- 知道各自的分离模式辅助逻辑在哪

---

## 18.1 同一套骨架，三种实例

Dynamo 对后端的集成模式是统一的（ch17 建立）：

```
main.py（角色装配）→ handlers/generate 循环 → publisher（事件+指标）
                                    ↘ 传输参数/分离辅助（引擎特化）
```

三个后端的差异集中在**引擎特化层**——Dynamo 侧的通用机制（发现、路由、
事件、KVBM）完全复用。这就是"编排层"的架构红利。

## 18.2 SGLang 集成

```
components/src/dynamo/sglang/
├── main.py                  # worker 入口
├── _disagg.py               # 分离辅助：bootstrap 地址计算、prefill 预热
├── engine_generate.py       # 引擎驱动
├── engine_routes.py         # 引擎路由面
├── request_handlers/        # llm / embedding / multimodal / diffusion
├── publisher.py             # 事件与指标
├── register.py              # 注册
└── sidecar.py               # Rust 边车挂载
lib/sidecar/sglang/src/      # Rust 边车
```

SGLang 特化点：

- **bootstrap 地址**（`_disagg.py::compute_bootstrap_address`）：SGLang 分离
  部署的握手寻址，Dynamo 把它接进 etcd 发现体系。
- **prefill 预热**（`warmup_prefill_engine`）：绕过冷启动首请求高延迟。
- 请求处理器按任务分族（`request_handlers/`），embedding/多模态/扩散模型
  是一等公民。
- 协议：`lib/llm/src/protocols/sglang/`（`Nv*` 类型）+ 独立 HTTP 表面
  （`lib/llm/src/http/service/sglang_generate.rs`）。

## 18.3 TensorRT-LLM 集成

```
components/src/dynamo/trtllm/
├── main.py                  # 入口
├── engine.py                # 引擎封装
├── engines/                 # 多引擎形态
├── request_handlers/        # 含 aggregated_handler（聚合分离收口）
├── publisher.py
├── sidecar.py
└── workers/                 # worker 形态
lib/sidecar/trtllm/src/      # Rust 边车
```

TRT-LLM 特化点：

- 引擎以编译产物（engine plan）为中心，Dynamo 侧管理其生命周期与批处理
  约定；
- `aggregated_handler.py` 处理聚合形态下分离语义的收口；
- 与 NVIDIA 硬件栈（NCCL/ NVLink 拓扑）配合最深——排障时 interconnect
  检查（ch15 的思路）对它尤其重要。

## 18.4 选型速查（从 Dynamo 视角）

| 维度 | vLLM | SGLang | TRT-LLM |
|------|------|--------|---------|
| 集成深度 | 最深（kv_connector/事件/omni） | 深（分离辅助、多任务族） | 深（引擎形态多） |
| 强项 | 社区/生态/长尾特性 | 前缀缓存（RadixAttention）、结构化输出 | NVIDIA 硬件上的极致性能 |
| 常见场景 | 通用首选 | Agent/结构化/多轮 | 追求 P99 的生产批量部署 |
| Dynamo 侧成熟度参照 | ch17 全套 | 本 章 | 本 章 |

**关键认知**：在 Dynamo 之下换引擎，frontend/router/KVBM 层零改动——改的
只是部署清单里 worker 组件的类型与参数（ch20 的 DGD 里就是一行组件规格）。

## 18.5 Mocker：第四种"后端"

`dynamo.mocker`（ch19 专题）实现了同一骨架的**零 GPU 版本**，也是所有
CI 的后端替身。读三个真实集成之前先读懂 mocker，能先把骨架与引擎语义
分离——这正是本 章"同构"论断的最好验证。

## 18.6 动手对照实验

1. `examples/backends/sglang/`、`examples/backends/trtllm/` 的 launch 脚本
   与 vLLM 的并排 diff：差异全在引擎参数，组件拓扑相同。
2. 同一负载分别跑 vLLM/SGLang worker（同 frontend 同 router），对比
   KV 命中率与 TTFT——验证"编排层不变，换引擎"的可操作性。

## 小结

- 集成骨架同构：入口/主循环/事件/边车；特化在传输参数、握手、任务族。
- SGLang 的 bootstrap/预热、TRT-LLM 的引擎形态是各自记忆点。
- 选型影响的是 worker 组件规格，不是 Dynamo 架构。

## 自检（3 题，自答）

1. "SGLang 的 bootstrap 地址计算"解决什么问题？为什么不能硬编码？
2. 换引擎时 frontend、router、KVBM 各需要改什么？（答案应当是"什么都不改，
   说出为什么"。）
3. 三个后端的 request_handlers 结构为何不同？差异来自 Dynamo 还是引擎？

## 下一步（跳转推荐）

- → [ch19 Mocker 与基准测试](ch19-mocker-bench.md)
- → [ch20 Operator 与 DGD](../06-deploy/ch20-operator-dgd.md)（多引擎在
  部署层的表达）
