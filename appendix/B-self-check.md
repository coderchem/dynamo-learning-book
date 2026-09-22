# 附录 B · 全书自检清单与常见误区

> 用法：像过检查单一样逐条自答。答不出的条目标注章节，回去重读。
> 面试备考生：把"误区"部分当作反直觉考点重点记忆。

## B.1 概念层（ch01–ch03）

- [ ] 说出 prefill/decode 各自的资源受限类型，以及由此推出的三种系统设计
      （chunked prefill、分离部署、独立扩缩容）。
- [ ] KV 字节公式默写，并解释每个因子。
- [ ] Dynamo 六大能力 + 各自源码归属（部分级即可）。
- [ ] 三层栈（Rust/Python/K8s）与三条通路（请求/事件/控制）的对应关系。
- [ ] 为什么 HTTP 服务器在 Rust 而 CLI 入口在 Python？

## B.2 运行时层（ch05–ch09）

- [ ] 手写一个两组件流水线（不抄书）：服务端 `serve_endpoint` + 客户端
      `client()` 流式消费。
- [ ] 三元组各字段语义；etcd 注册在哪个调用里发生。
- [ ] 四种传输的用途矩阵；NATS `max_payload` 的现实约束。
- [ ] worker 生命周期状态机（含 `kill -9` 路径），lease 的角色。
- [ ] `ModelWatcher` 的两个输出（路由表、tokenizer 映射）为什么重要。
- [ ] 请求八阶段按序复述，每阶段给出文件路径。

## B.3 路由层（ch10–ch13）

- [ ] 一条 KV 事件从 vLLM 引擎到 radix tree 的完整通路（含 MM 归一化、
      PlacementEvent 的 tier、批处理/去重、事件源的 etcd 发现注册）。
- [ ] "索引允许错、结果不允许错"——解释这个性质如何支撑整条链路的优化，
      并举出缺口检测/引擎重启/多记/少记各自的机制。
- [ ] `RawKvEvent::BlockStored` 的字段清单里 `lora_name`、`cache_salt`、
      `parent_block_hash` 各自为什么存在。
- [ ] 块哈希身份 = Blake3 链接哈希 + 哪三重盐？keyed tracking 防什么攻击？
- [ ] `SyncIndexer::find_matches` 的输入输出；`early_exit` 的意义。
- [ ] `dump_tree_as_events` 为何足以充当快照机制。
- [ ] **默写 ch12 打分公式**：缓存抵扣三层计价（1.0/0.75/0.25）、热点
      衰减、decode 特例、taints 乘子、T=0 蓄水池破平与 T>0 softmax 采样。
- [ ] 路由模式全集（7 种）与模式降档排障法；`DYN_ROUTER_*` 关键旋钮与默认值。
- [ ] 两个"调度"（引擎内 vs Dynamo 路由）的边界。
- [ ] FCFS vs WSPT 各优化什么目标函数？`router_queue_threshold` 何时触发排队？
- [ ] 序列账本的生命周期方法；`router_track_output_blocks` 解决什么低估。
- [ ] 准入（TinyLFU）与驱逐（fifo/lru/multi_lru/lineage）为什么是正交决策。

## B.4 分离与 KVBM（ch14–ch16）

- [ ] 分离三阶段时序图 + `WorkerType` 四角色。
- [ ] 条件分离解决什么；短请求为什么绕过。
- [ ] 一次 KV 迁移的两端协作五步；字节数估算。
- [ ] `_DeferredAbort` 为什么存在。
- [ ] KVBM 三层（logical/physical/engine）的职责分离与 OS 类比。
- [ ] 下放与拉回两条控制流的触发条件与事件闭环。
- [ ] 新旧两代块管理器的路径辨识。

## B.5 后端与部署（ch17–ch21）

- [ ] vLLM worker 启动五步；三个关键钩子（传输参数/abort 防护/kv_hints）。
- [ ] "命中了但没省"的三点一致性检查。
- [ ] 三引擎同构集成骨架；选型差异。
- [ ] mocker 的三个用途与其结论边界。
- [ ] DGD → Pod 的渲染管线；Grove/EPP 的插入点。
- [ ] Planner 决策链与"改声明不动机器"原则。
- [ ] 排障地图：七个症状 → 对应章。

## B.6 常见误区（反直觉考点）

1. **"Dynamo 是推理引擎"** —— 不是。不执行前向计算，不含 attention kernel；
   它编排引擎（ch02）。
2. **"用 `dynamo serve` 启动"** —— 0.x 时代命令，1.5.0 已移除。入口是
   `python -m dynamo.<component>`（ch02/ch05）。
3. **"协议类型在 `lib/api`"** —— 无此 crate；协议来自外部
   `dynamo-protocols`/`dynamo-parsers`（ch03/ch04）。
4. **"dynamo.frontend 是 Python HTTP 服务"** —— Python 只是入口壳，服务器
   在 `lib/llm/src/http/`（Rust）（ch09）。
5. **"KV 索引错了会答错题"** —— 不会。索引只影响选哪个 worker；发错了
   代价是重算（次优），不是错误答案（ch10）。
6. **"Dynamo 管 continuous batching"** —— 不管。引擎内调度归引擎
   （ch13）。
7. **"分离部署永远更优"** —— 短请求/小规模/差网络下反而更差；有条件分离
   机制存在的原因（ch14）。
8. **"扩容 = 创建 Pod"** —— Planner 改 DGD，Operator 调和，etcd 发现跟上，
   三步之后流量才迁移（ch20/21）。
9. **"命中率低 = 路由坏了"** —— 先查负载前缀重复度和事件平面，再查路由
   逻辑（ch21 排障地图）。
10. **"mocker 上验证的性能结论可以直接上生产"** —— 凡涉及物理代价（KV
    迁移带宽、真实计算）的结论必须真实环境复测（ch19）。

## B.7 读码能力终极检验

打开仓库，不看本书，完成：

1. 从 `components/src/dynamo/frontend/main.py` 出发，列出 `run_input("http")`
   之后调用链上的前五个 Rust 函数所在文件。
2. 在 `lib/kv-router/src/indexer/radix_tree.rs` 里找到查询入口，说明它
   的输入输出类型。
3. 在 `deploy/operator/internal/controller/` 里指出组件副本数最终落到
   哪个对象的哪个字段。

三项都能独立完成，这本书的目标就达到了。
