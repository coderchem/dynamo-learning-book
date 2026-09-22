# 附录 A · 术语表

> 按主题分组。中文读者请同时记住英文原词——代码、日志、指标里只有英文。

## 运行时与通信

| 术语 | 英文 | 一句话 |
|------|------|--------|
| 组件 | Component | 一个 `python -m dynamo.<x>` 进程，部署单元（ch06） |
| 端点 | Endpoint | 组件内的可调用入口，三元组 `namespace.component.endpoint`（ch06） |
| 命名空间 | Namespace | 部署边界，注册 key 顶层前缀（ch06） |
| 请求平面 | request plane | 组件间请求/token 流通路：TCP/ZMQ/NATS（ch07） |
| 事件平面 | event plane | KV 事件广播通路，可插拔 NATS/ZMQ（ch07） |
| 控制面 | control plane | etcd 承载的注册/发现/锁（ch07） |
| 服务发现 | service discovery | etcd lease 注册 + watcher 监视 → 路由表（ch08） |
| 租约 | lease | etcd 保活机制，进程死的注册自动消失（ch08） |

## 请求处理

| 术语 | 英文 | 一句话 |
|------|------|--------|
| 预填充 | prefill | 处理 prompt 的计算阶段，算力受限（ch01） |
| 解码 | decode | 逐 token 生成阶段，带宽受限（ch01） |
| 首 token 延迟 | TTFT, time to first token | 客户端视角的核心 SLO 之一 |
| 每 token 延迟 | ITL/TPOT | 生成期延迟 |
| 前处理器 | preprocessor | chat template/媒体/工具的请求改写（ch09） |
| 聚合器 | aggregator | token 流 → OpenAI SSE 响应的收口（ch9） |
| Nv 请求类型 | `NvRequest` 等 | `lib/llm/src/protocols/` 的内部协议类型 |

## KV 与路由

| 术语 | 英文 | 一句话 |
|------|------|--------|
| KV 缓存 | KV cache | 注意力历史状态，推理核心负载状态（ch01） |
| KV 事件 | KV event | worker 广播"我缓存/释放了哪些块"（ch10） |
| 前缀索引 | prefix indexer | radix tree 族，回答"谁有这个前缀"（ch11） |
| 基数树 | radix tree | 按块压缩的前缀树 |
| 布谷过滤器 | cuckoo filter | 空间高效的成员近似查询（跨 DC 场景）（ch11） |
| 块哈希 | block hash | Blake3 块身份，跨 worker 一致（ch10） |
| KV 感知路由 | KV-aware routing | 按前缀命中+负载选 worker（ch12） |
| 路由约束 | `RoutingConstraints` | 请求级硬过滤（角色/亲和/迁移）（ch12） |
| 序列跟踪 | sequence tracking | 请求在途状态，支撑容错与接力（ch13） |
| 驱逐 | eviction | 释放 KV 块：FIFO/LRU/lineage（ch13） |
| 下层索引 | lower-tier indexer | 记录 KV 在 CPU/SSD 层的索引变体（ch11/16） |

## 分离与传输

| 术语 | 英文 | 一句话 |
|------|------|--------|
| 分离式服务 | disaggregated serving (PD disagg) | prefill/decode 独立部署（ch14） |
| 接力 | handoff / activation | prefill 完成 → decode 接管的协议（ch14） |
| 工作角色 | `WorkerType` | Prefill/Decode/Encode/Aggregated（ch14） |
| 条件分离 | conditional disaggregation | 按请求特征决定是否走分离（ch14） |
| NIXL | NIXL | NVIDIA 传输抽象库：RDMA/NVLink/UCX（ch15） |
| 传输执行器 | transfer executor | nixl/cuda/memcpy 三兄弟（ch15） |
| 引导地址 | bootstrap address | SGLang 分离握手寻址（ch18） |

## KVBM

| 术语 | 英文 | 一句话 |
|------|------|--------|
| KV 块管理器 | KVBM (KV Block Manager) | `lib/kvbm-*` 多层 KV 管理栈（ch16） |
| 逻辑层 | logical layer | 块记账与驱逐策略（ch16） |
| 物理层 | physical layer | 布局与搬运（ch16） |
| 下放/上载 | offload / onboard | GPU↔CPU/SSD 的层间移动（ch16） |
| 谱系驱逐 | lineage eviction | 按序列家谱价值的驱逐（ch13/16） |

## 部署与运维

| 术语 | 英文 | 一句话 |
|------|------|--------|
| DGD | DynamoGraphDeployment | 描述推理组件图的 K8s CRD（ch20） |
| Operator | operator | `deploy/operator`，把 DGD 渲染成工作负载（ch20） |
| Grove | Grove | 拓扑感知 gang 调度 operator（ch20） |
| 推理网关 | Inference Gateway | Envoy ext-proc 入口层（ch20） |
| Planner | planner | SLA 驱动扩缩容大脑（ch21） |
| 性能模型 | perf model | 负载 → SLO 的预测模型（ch21） |
| Mocker | mocker | 可编程假引擎（ch19） |

## 生态仓库名

NIXL / AIPerf / AISimulate / ModelExpress / Grove —— 见 ch01 §1.6 表。
