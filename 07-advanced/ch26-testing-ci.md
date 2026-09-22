# 第 26 章 · 测试体系与 CI

> **适合谁读**：所有要给 Dynamo 提 PR 的人；想搭推理服务测试体系的团队。
> **前置**：ch05（本地环境）。
> **耗时**：35 分钟
> **源码锚点**：commit `61e9184`（v1.5.0）。

**学完能：**
- 按目录找到每类测试并跑起来
- 用 `run_serve_deployment` 一把梭写一个新引擎/新配置的端到端用例
- 说清 CI 的门控模型（marker、GPU 档位、copy-pr-bot）与提 PR 前的自查清单

---

## 26.1 测试地图

```
tests/
├── serve/               # 端到端：真实拉起组件进程（核心是 common.py 的 harness）
├── router/              # 路由集成（独立 router 服务形态）
├── frontend/            # 前端行为
├── runtime/             # Rust runtime 的 Python 侧集成
├── kvbm_integration/    # KVBM 栈集成
├── fault_tolerance/     # 容错规格（ch24 已详述）
├── deploy/              # 部署形态
├── lmcache/  gpu_memory_service/   # 第三方/子系统专项
├── basic/  parity/      # 基础与一致性
Rust 侧：lib/*/tests/、模块内 #[cfg(test)]（如 indexer/tests.rs、
        selector/default.rs 内嵌测试、recovery/cursor.rs 内嵌测试）
```

分层哲学：**Rust 单测钉算法边界（同分随机、游标四态、taints 过滤），
Python 集成钉系统行为（发现、路由、流式、容错）**——你在 ch11/12/24
读到的每个"单测叫什么"都来自前一层。

## 26.2 核心 harness：`run_serve_deployment`

`tests/serve/common.py:338` 是所有后端端到端测试的公共入口：

```python
def run_serve_deployment(
    config: EngineConfig,
    request: Any,
    *,
    ports: ServicePorts | None = None,   # 传 dynamo_dynamic_ports
    extra_env: Optional[Dict[str, str]] = None,
    post_validation: Optional[Callable[[], None]] = None,
) -> None:
    """Run a standard serve deployment test for any EngineConfig.
    - Launches the engine via EngineProcess.from_script
    - Builds a payload (with optional override/mutator)
    - Iterates configured endpoints and validates responses and logs
    - Optionally runs a final assertion while the deployment is still alive
    """
```

四步流水：**起部署（脚本拉起真实组件进程）→ 发请求（`request_payloads`，
必填）→ 校验响应与日志 → 存活期内跑 `post_validation`**。

给新引擎/新配置加用例的最短路径：构造一个 `EngineConfig`（模型、
`script_name`、请求载荷、要打的端点列表），复用这个函数——不要自己
拼进程管理。两个细节：

- `ports` 传 `dynamo_dynamic_ports`：并行跑的用例不抢固定端口。
- `extra_env`：注入被测旗标（如 `DYN_ROUTER_TEMPERATURE`），比改脚本干净。

日志校验是 harness 的一等公民（"validates responses **and logs**"）——
`tests/utils/router_logs.py` 解析 ch12 讲过的 `[ROUTING] Best: ...` 结构化
日志做断言：**路由行为的回归测试靠日志契约，不靠反编译内部状态**。

## 26.3 marker 与 GPU 门控

`pyproject.toml` 的 `[tool.pytest.ini_options]` 定义严格 marker
（`--strict-markers`，拼错即失败），包括 GPU 分档 `gpu_0 … gpu_8`：
按"需要几张卡"给用例分级，CI 按档位调度。日常：

```bash
pytest -m unit tests/          # 单测（无 GPU）
pytest tests/serve -k vllm     # vLLM 端到端（按 marker/机器能力）
cargo test -p dynamo-kv-router # Rust 单 crate
```

写测试前读两份规范：`.ai/pytest-guidelines.md`（marker、fixture 约定）与
`.ai/test-model-size-guardrails.md`（测试模型的大小红线——CI 资源有限）。

## 26.4 Rust 侧的测试品味

三个已读过的样本展示三种风格：

| 风格 | 样本 | 钉住的行为 |
|------|------|-----------|
| 算法边界 | `selector/default.rs::test_default_selector_randomizes_zero_temperature_ties` | T=0 同分必须随机破平 |
| 状态机规格 | `recovery/cursor.rs::live_observation_detects_contiguous_gap_and_stale_ids` | 游标四态一字不差 |
| 集成行为 | `indexer/tests.rs` | 事件应用/worker 摘除/修剪 |

给路由改代码的 PR 自查：公式改动 → default.rs 补打分边界用例；新事件
语义 → indexer 用例 + 若影响恢复则 cursor 用例；端到端 → serve harness。

## 26.5 CI 门控模型

来自仓库根 `AGENTS.md`（CI 侧的权威约定）：

1. **pre-commit 全量**：`pre-commit run --all-files`（含 DCO、生成物
   校验、skills 校验等钩子）——本地先过，别浪费 CI 排队。
2. **fork PR 信任链**：完整 CI 只在维护者 `/ok to test <sha>` 后由
   copy-pr-bot 建 `pull-request/N` 分支触发；fork PR 要自动获批还需
   每个提交 GitHub `Verified` 签名（DCO 签-off 不算）。
3. **CODEOWNERS 100% 覆盖**：生成物，改 `.github/codeowners/areas.yaml`
   再生成。
4. 排障入口：`.ai/ci-guidelines.md` 与 `pr-monitor` 技能（按 PR 查 CI
   健康、区分 PR 引入 vs main 上就有的 flaky）。

## 26.6 动手：为一个路由改动写全套验证

以"给 ch12 的打分加一个新权重项"为例：

1. `default.rs`：单测——权重为 0 时行为与旧版逐位一致；权重非 0 时
   期望的偏好方向。
2. `serving/config.rs`：配置解析/默认值/环境变量的解析用例。
3. `tests/serve/`：mocker 拓扑 + `extra_env` 设新权重，`post_validation`
   里用 `router_logs.py` 断言选择分布变化。
4. 本地 `pre-commit run --all-files` + `cargo test -p dynamo-kv-router` +
   `pytest tests/serve -k <你的用例>`，然后才提 PR。

## 小结

- 分层：Rust 单测钉算法、Python harness 钉系统；日志契约是路由回归的
  载体。
- `run_serve_deployment` 四步流水是新增端到端用例的唯一正道。
- 严格 marker + GPU 分档 + `/ok to test` 门控：本地全绿再交 CI。

## 自检（3 题，自答）

1. 为什么路由行为的断言靠结构化日志而不是读路由内部结构？
2. `gpu_3` marker 表达什么？它如何影响 CI 调度？
3. 你的 fork PR 提交只有 DCO sign-off，自动信任 CI 为什么不会触发？

## 下一步（跳转推荐）

- → [附录 B 全书自检清单](../appendix/B-self-check.md)
- → 回 [ch05 本地跑起来](../01-overview/ch05-getting-started.md) 把测试
  环境补齐
