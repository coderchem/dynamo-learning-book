# 第 23 章 · LoRA 适配器管理

> **适合谁读**：要多租户/多风格微调共用一个基座模型的平台工程师。
> **前置**：ch10（块哈希）、ch12（路由输入）。
> **耗时**：40 分钟
> **源码锚点**：commit `61e9184`（v1.5.0）。

**学完能：**
- 解释 LoRA 身份如何进入块哈希，为什么这决定了缓存正确性
- 列出 `lib/llm/src/lora/` 子系统的分工（下载/缓存/准入/预测）
- 部署一个多 LoRA 服务的知道要配什么、验什么

---

## 23.1 问题：一个基座，N 个适配器

LoRA 让"每个租户一个微调模型"变成"一个基座 + N 个小适配器"（每个几十
到几百 MB）。但 KV cache 是**适配器敏感**的：同样的 token 过不同的
适配器，算出的 K/V 不同。于是三件事必须做对：

1. **身份隔离**：适配器 A 的缓存块绝不能被适配器 B 的请求命中。
2. **物料管理**：适配器从哪来（HF/S3/本地）、怎么缓存、怎么校验。
3. **容量准入**：GPU 上同时能驻留几个适配器、该换入换出谁。

Dynamo 对应的三层实现都在 `lib/llm/src/lora/`（Rust 核心）加
`lib/bindings/python/rust/llm/lora.rs`（Python 绑定）。

## 23.2 身份隔离：LoRA 进入块哈希

`lib/kv-router/src/indexer/README.md`（FlashIndexer 文档）给了准确定义：

- **LocalBlockHash** 在块内 token 之外，可选地混入 LoRA 适配器名——
  长度前缀后拼接再哈希（`hash(tokens || len("my-lora") || "my-lora")`），
  保证基座与各适配器的同 token 块**哈希必然不同**。
- **引擎侧负责**：`ExternalSequenceBlockHash` 由引擎计算上报，LoRA 身份
  由引擎在哈希时混入——vLLM 的路径是 `_gen_lora_extra_hash_keys`，把
  LoRA ID 作为 `hash_block_tokens(..., extra_keys)` 的附加键。Dynamo 路由
  层不重复添加。
- **线格式携带**：ch10 的 `RawKvEvent::BlockStored` 有 `lora_name` 字段；
  ch12 的 `BestMatchArgs` 也带 `lora_name`——路由查询在适配器域内进行。

> **正确性含义**：隔离失败不是"慢"，是**答错**（B 租户读到 A 租户适配器
> 算的 KV）。这就是为什么身份必须在哈希层解决，而不是路由层的软过滤。

与 ch11 的第三重盐（keyed tracking）对照：LoRA/命名空间盐防**误共享**，
keyed tracking 防**外部构造哈希探测**——三重盐各管一件事。

## 23.3 物料管理：下载与缓存

Python 绑定暴露统一入口 `LoRADownloader`（`lora.rs`，99 行，可通读）：

```python
from dynamo.llm import LoRADownloader
dl = LoRADownloader(cache_path=None)      # 默认从环境变量取缓存目录
path = await dl.download_if_needed(lora_uri)   # 返回本地路径
dl.is_cached(lora_uri); dl.validate_cached(key); dl.uri_to_cache_key(uri)
```

- 三种 URI：`file://`（原路径直用不拷贝）、`s3://`（下载进缓存）、
  `hf://`（下载进 HuggingFace snapshot 缓存）。Rust 侧
  `source.rs` 定义 `LocalLoRASource / HuggingFaceLoRASource / S3LoRASource`。
- `cache.rs` 的 `uri_to_cache_key` 保证 **Rust 与 Python 两侧生成一致的
  缓存键**；`validate_cached` 校验缓存目录里必需文件齐全——冷启动拉到
  半截的适配器在加载前就会被拦下。

## 23.4 容量准入：控制子系统

`lib/llm/src/lora/` 的其余文件构成一个完整的**多 LoRA 准入控制**子系统：

```
controller.rs       # 控制器：协调下面的组件
predictor.rs        # 预测：适配器的未来需求（热度/请求趋势）
load_estimator.rs   # 负载估计：驻留成本核算
filter.rs           # 过滤：当前能服务哪些适配器
state_tracker.rs    # 状态：各 worker 上适配器的驻留状态
routing/            # 适配器感知的路由子模块
```

设计意图与 ch13 的驱逐问题同构，但对象换成了适配器：GPU 显存里
"热适配器集合"的换入换出 = 一个小型缓存管理问题（预测将来谁会被用到、
现在谁最不值得驻留）。与 KV 驱逐不同的是，换错适配器的代价是一次加载
延迟（秒级），而不是答错——所以这里的策略可以更激进。

## 23.5 与前文的连接点

| 机制 | LoRA 下的变化 | 章 |
|------|--------------|-----|
| 块哈希 | 混入长度前缀的适配器名（引擎侧） | ch11 |
| 事件线格式 | `lora_name` 字段 | ch10 |
| 路由输入 | `BestMatchArgs.lora_name`、`cache_namespace` | ch12 |
| 命名空间盐 | 另一个独立的隔离维度（租户级） | ch11 |

排障口诀：**多 LoRA 部署命中率异常低**，先查三处——引擎版本是否带
`_gen_lora_extra_hash_keys`（老 vLLM 不混 LoRA 进哈希）、前端与 worker
的适配器名是否完全一致（大小写/别名）、`cache_namespace` 是否把本该共享
的域切开了。

## 23.6 动手

1. 通读 `lib/bindings/python/rust/llm/lora.rs`（99 行）——这是" bindings
   如何薄封装 Rust 核心"的最佳小样本。
2. mocker + 两个虚拟 LoRA 名发同前缀请求，观察索引里是否形成两条
   独立的命中链（日志里 lora 域隔离）。
3. 思考：如果要在 ch12 的打分公式里加"适配器已在目标 worker 驻留"的
   抵扣项，应该加在哪一项？（对照 `decode_active_request_weight` 的
   建模方式）

## 小结

- LoRA 身份在**哈希层**隔离（引擎混入、路由域内查询），隔离失败 = 答错。
- `LoRADownloader` 统一 file/s3/hf 三源下载与校验；`lib/llm/src/lora/`
  的 controller/predictor/load_estimator/filter/state_tracker 是完整的
  驻留准入子系统。
- 多 LoRA 命中异常排查三查：引擎哈希、名称一致性、命名空间。

## 自检（3 题，自答）

1. 为什么 LoRA 必须混进块哈希而不能只在路由层过滤？两种方案在事件丢失
   （ch10 尽力而为）时行为有何不同？
2. 适配器换入换出与 KV 块驱逐（ch13）在"决策错误的代价"上有什么本质
   区别？这如何影响策略激进度？
3. `uri_to_cache_key` 为什么要同时存在于 Rust 和 Python 两侧？

## 下一步（跳转推荐）

- → [ch24 容错、迁移与恢复](ch24-fault-tolerance.md)
