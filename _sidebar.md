- [首页 · 总览与学习路径](/README.md)

- **第一部分 入门与架构**
  - [ch01 预备知识：LLM 推理与引擎生态](/01-overview/ch01-llm-inference-primer.md)
  - [ch02 Dynamo 是什么：定位与核心能力](/01-overview/ch02-what-is-dynamo.md)
  - [ch03 总体架构：三层栈](/01-overview/ch03-architecture.md)
  - [ch04 仓库地图与代码地标](/01-overview/ch04-repo-map.md)
  - [ch05 本地跑起来](/01-overview/ch05-getting-started.md)

- **第二部分 核心抽象**
  - [ch06 Runtime：组件、端点与命名空间](/02-core/ch06-runtime-components.md)
  - [ch07 传输四平面：etcd / NATS / TCP / ZMQ](/02-core/ch07-transports.md)
  - [ch08 服务发现与 Worker 生命周期](/02-core/ch08-discovery.md)
  - [ch09 前端入口：一次 HTTP 请求的完整旅程](/02-core/ch09-frontend-entry.md)

- **第三部分 KV 感知路由**
  - [ch10 KV 事件与发布订阅](/03-kv-routing/ch10-kv-events.md)
  - [ch11 前缀索引：Radix Tree 与变体](/03-kv-routing/ch11-prefix-index.md)
  - [ch12 路由决策与 Worker 选择](/03-kv-routing/ch12-routing-decision.md)
  - [ch13 调度、驱逐与序列跟踪](/03-kv-routing/ch13-scheduling-eviction.md)

- **第四部分 分离式服务与 KV 传输**
  - [ch14 PD 分离架构](/04-disagg/ch14-pd-disagg.md)
  - [ch15 NIXL 与 KV 传输](/04-disagg/ch15-nixl-transfer.md)
  - [ch16 KVBM：多层 KV 管理](/04-disagg/ch16-kvbm.md)

- **第五部分 后端集成**
  - [ch17 vLLM 后端集成](/05-backends/ch17-vllm-backend.md)
  - [ch18 SGLang 与 TensorRT-LLM 后端](/05-backends/ch18-sglang-trtllm.md)
  - [ch19 Mocker 与基准测试](/05-backends/ch19-mocker-bench.md)

- **第六部分 部署与运维**
  - [ch20 Kubernetes Operator 与 DGD](/06-deploy/ch20-operator-dgd.md)
  - [ch21 Planner 与可观测性](/06-deploy/ch21-planner-observability.md)

- **附录**
  - [A 术语表](/appendix/A-glossary.md)
  - [B 全书自检清单与常见误区](/appendix/B-self-check.md)
