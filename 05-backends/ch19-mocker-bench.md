# 第 19 章 · Mocker 与基准测试

> **适合谁读**：所有要做性能验证、容量规划或 CI 的读者。
> **前置**：ch05、ch17（骨架概念）。
> **耗时**：35 分钟
> **源码锚点**：commit `61e9184`（v1.5.0）。

**学完能：**
- 用 Mocker 配置出可控的假负载（延迟曲线、TTFT/ITL 形态）
- 跑 frontend 基准并解读输出指标
- 区分"压前端"与"压全链路"两种实验的设计差异

---

## 19.1 Mocker 是什么

一个**行为可编程的假推理引擎**，实现了 ch17 骨架的全部契约，但没有模型：

```
lib/mocker/                          # Rust 引擎本体
├── src/...                          # 引擎逻辑 + vLLM/SGLang 协议 mock
└── servers/{vllm,sglang}/src/main.rs  # 独立协议级 mock 服务器
components/src/dynamo/mocker/
├── main.py        # 入口（main() 286 行，已核验；--is-prefill-worker 支持分离）
├── config.py      # 行为配置
├── args.py
└── aic_session.py
lib/llm/src/mocker.rs               # LLM 侧胶水
```

它能模拟：

- **时间形态**：TTFT 分布、每 token 间隔（ITL）曲线——按 prompt 长度等
  条件缩放；
- **角色**：聚合 / prefill / decode（分离拓扑也能无 GPU 复现）；
- **协议**：vLLM/SGLang 格式的应答（`servers/`）。

## 19.2 为什么 Mocker 重要

1. **CI 的后端**：`tests/serve/` 大量用例不依赖 GPU 也能验证编排正确性。
2. **性能实验的变量隔离**：把"引擎速度"钉死，只测 Dynamo 自己的开销
   （路由延迟、事件开销、HTTP 层）。
3. **容量规划的负载模型**：`lib/data-gen/` 提供合成 trace（含 weka/satf/
   mooncake 模型），喂给 mocker 化的拓扑，可在无 GPU 集群上逼近真实行为
   （与 AISimulate 的离线预测互补，见 ch21）。

## 19.3 前端基准

仓库自带的 frontend 基准设施（`benchmarks/frontend/`、`benchmarks/mocker/`，
配套 `.agents/skills/dynamo-frontend-benchmark` 的方法论）：

- 拓扑：`dynamo.frontend` + N 个 mocker worker；
- 变量：并发、输入/输出长度分布、流式与否、路由模式；
- 测量：吞吐、P50/P99 TTFT、P50/P99 ITL、frontend CPU 分布。

典型实验设计（对照法）：

| 问题 | 固定什么 | 扫什么 |
|------|----------|--------|
| 路由开销多大 | mocker 行为、负载 | router-mode（kv vs rr） |
| 事件平面影响 | 负载、worker 数 | KV 事件开关/批量参数 |
| 扩 worker 收益 | 每 worker 负载曲线 | worker 副本数 |

> 读数纪律：无 GPU 环境的结论只对"编排层"有效；一旦结论要外推到真实
> 推理（比如"分离更优"），必须回到真实引擎复测——mocker 没有 KV 迁移的
> 物理代价（除非专门配置模拟，ch15 的字节数估算仍然要手算）。

## 19.4 全链路基准

真实后端 + 真实负载：

- 负载生成：生态里的 [AIPerf](https://github.com/ai-dynamo/aiperf)（K8s
  内压测，仓库技能链 `configure-aiperf-benchmark` / `run-aiperf-benchmark` /
  `analyze-aiperf-results` 即其方法论），或 `lib/bench/` 下的 Rust 工具
  （trace 重放类：`request_trace_to_mooncake` 等 `[[bin]]`）。
- SLO 剖析：`components/src/dynamo/profiler/`（`profile_sla.py`、
  `rapid.py`/`thorough.py` 两档）。
- 请求 trace：`benchmarks/` 与 `lib/data-gen/`（合成）。

分层测量的顺序建议：先 mocker 验证编排开销可忽略 → 再真实引擎单 worker
定基线 → 最后全拓扑扫参。**跳过第一层直接扫参，出了问题分不清是 Dynamo
还是引擎**。

## 19.5 动手：一个完整的对照实验

```bash
# 1. 基线：mocker 聚合 + round-robin
python -m dynamo.frontend --model mock/model &
python -m dynamo.mocker --model mock/model &
# 压测记录 TTFT/ITL/吞吐（用你熟悉的压测工具打 /v1/completions）

# 2. 切 kv 模式，同一负载（前缀重复度高的对话型 trace）
#    观察：TTFT 下降幅度 vs 索引开销（ITL 应基本不变）

# 3. 加到 4 个 mocker worker，观察路由分布是否均衡（ch12 的负载项）
```

## 小结

- Mocker = 可编程假引擎：CI 后端、变量隔离器、无 GPU 容量实验的载体。
- 前端基准测"编排开销"，全链路基准（AIPerf/profiler）测"业务 SLO"；
  分层进行，先隔离后复合。
- mocker 结论不得外推到物理代价相关的命题（KV 迁移带宽）。

## 自检（4 题，自答）

1. 为什么 mocker 能验证"分离编排正确"却不能验证"分离性能更优"？
2. 设计实验测"KV 事件平面在大集群的 CPU 开销"：固定什么、扫什么、看什么指标？
3. `lib/mocker/servers/vllm` 与 `dynamo.mocker` 组件的区别是什么？各自适用？
4. 你的 mocker 实验 TTFT 降了 30%，向老板汇报前还欠什么验证？

## 下一步（跳转推荐）

- → [ch21 Planner 与可观测性](../06-deploy/ch21-planner-observability.md)
- → 回 [ch12 路由决策](../03-kv-routing/ch12-routing-decision.md) 做实验闭环
