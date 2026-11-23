# 扩展 HBFSim

> 状态：当前版本（Current）  
> 最近审核日期：2026-09-29

HBFSim 的设计目标是让大多数新想法无须修改物理模型。本指南按工作量从小到大排列扩展点；请从最上方开始，只有当前方式无法满足需求时才继续深入。

| 你想要…… | 扩展点 | 语言 | 是否修改引擎？ |
| --- | --- | --- | --- |
| 尝试其他硬件参数 | 配置 Overlay | `key=value` | 否 |
| 研究另一种工作负载 | Trace 文件或生成器 | 任意语言 / Python | 否 |
| 测试放置、缓存、迁移或预取想法 | 基于 Session API 的 Python 策略 | Python | 否 |
| 发布用于 Trace 重放的策略 | 参考策略 | C++ | 否（与引擎链接） |
| 添加报告或指标 | Summary 读取器 | Python | 否 |
| 建模新的物理机制 | 核心引擎 | C++ | 是——请先阅读下方规则 |

## 1. 配置

配置文件由带 `#` 注释的 `key=value` 行组成，并应用在 System Profile 之后。Key 就是不带 `--` 的长选项名，因此可先用 `--set KEY=VALUE` 试验任意文件配置。

```ini
# my-overlay.cfg — 更快的 NAND 裸片和更大的写缓冲。
hbf-read-ns=2000
hbf-program-ns=50000
hbf-write-buffer-pages=4096
```

```bash
python3 -m hbfsim run --overlay path/to/my-overlay.cfg
```

个人 Overlay 可以放在任何位置。若要将其发布到 `configs/`，请在文件头注释中说明它隔离了什么变量及数值来源，将它列入[配置目录](../../configs/README.md)，并为每个**新增**物理参数在 `configs/parameter-provenance.json` 中添加条目，否则 `parameter_provenance_coverage` 测试会失败。用于补足未公开规范的数值可以接受，但必须标记为 `exploratory_assumption`。

## 2. 工作负载

参考运行器读取文本 Trace，每行一条内存操作：

```text
<address> <R|W> [bytes] [kind] [phase=N] [layer=N] [compute_ns=N] [at=NS]
```

- `kind` 可取 `model_weights`、`shared_context`、`generated_context`、`scratch`、`metadata`；语义策略会使用它，例如 `direct-read` 把可变数据类型保留在 HBM 中。
- `layer=N` 按执行顺序标记 Layer，且不可递减；Streaming 策略会在运行 N 时预取 N+1。要么标记全部请求，要么都不标记。
- `at=NS` 指定到达时间；省略时，请求按 `interarrival-ns` 间隔到达。

[示例 4](../../examples/04_custom_trace.py)展示如何用 Python 编写 Decode Trace。`workloads/synthetic/generate.py` 可生成带明确读写比例的顺序、随机、跨步和热点模式；[工作负载目录](../../workloads/README.md)介绍局部性分析和质量检查。对于支持连续批处理与 Paged KV Cache 的 LLM Serving，请使用直接驱动引擎的 [ServeLoop](serveloop.md)。

## 3. Python 策略

引擎可执行任意事务 DAG。策略代码为每个应用访问决定要发出哪些事务：

```python
import hbfsim
from hbfsim_client import Transaction

with hbfsim.open_session("4hbm-4hbf", overlays=["ocp-v070-grade2"],
                         initial_hbf_logical_pages=1024) as session:
    miss = Transaction(id="miss", target="HBF_LOGICAL", op="R",
                       addr=0x2000, bytes=4096, issue_ns=0.0)
    fill = Transaction(id="fill", target="HBM", op="W", addr=0,
                       bytes=4096, issue_ns=0.0, dependencies=("miss",))
    result = session.run([miss, fill])
    print(result.completion("fill").finish_ns)
```

- **Target** 列于[概念指南](../concepts_CN.md#事务-target)：HBM、经 FTL 的 HBF、静态或原始 Flash、基底裸片链路复制、外部后端、直连通道和 Barrier。
- **依赖关系**规定事务顺序：事务在全部依赖完成后启动，但不会早于自身的 `issue_ns`。
- **Batch** 是提交单位。`session.run(...)` 会为其编号；默认情况下每个事务都属于约束下一 Batch 的 `frontier`，后续 Batch 需要依赖的 ID 必须用 `retain` 保留。在保留集合内，ID 必须唯一。
- **结果**包含每个事务的到达、开始和完成时间，`result.receipt` 中按 Target 分类的延迟统计，以及累计设备状态。
- `open_session` 解析 Profile 名称，把 `options` 写入 `overrides.cfg`，并将磨损报告保存在 Session 的 `out/` 目录。`initial_hbf_logical_pages=N` 表示启动时已有 N 个逻辑 Page 写入。

[示例 5](../../examples/05_custom_policy.py)实现了带 LRU 淘汰和 Hazard-safe Fill 的完整晋升 Cache。策略只选择事务，因此它与其他结果使用相同的物理模型。协议详见 [hbfsim_client](../../hbfsim_client/README.md) 和[可观测性参考文档](../reference/observability.md)。

## 4. C++ 参考策略

当一个策略需要供所有人重放 Trace 时，可将它加入 `src/policies/reference/` 中的参考套件：

1. 在 `direct_policy`、`layer_streaming_policy` 和 `behavioral_tiering_policy` 旁实现策略，只依赖 `hbfsim_core` 与 `policies/policy_common`；核心模块绝不能包含策略。
2. 在 `src/app/reference_runner.cpp` 中现有的 `k…Scenario` 常量及 Dispatch Table 旁注册场景名称。
3. 在 `tests/cpp/` 下添加聚焦测试，并在 `cmake/TestTargets.cmake` 和某个 `cmake/tests/*.cmake` 分组中注册。
4. 在文档门禁检查的所有位置记录该场景，包括 README 表格、[场景参考](../reference/scenarios.md)、[模型](../reference/model.md)、[可观测性](../reference/observability.md)、[项目布局](../project-layout.md)，并加入 `hbfsim/catalog.py`，使统一入口可以列出它。

## 5. 报告和指标

每次运行都会写出 `summary.json`（Schema 为 `hbfsim.simulation.summary`）。使用 `hbfsim.load_summary(path)` 读取；每个场景都暴露统一入口的核心指标和完整原始 Row。添加核心指标时，在 `hbfsim/results.py` 中追加一个 `Metric`；通用 Renderer 应放在 [`reports/`](../../reports/README.md)，特定研究的图表则应和实验放在一起。

## 6. 物理机制

修改 `src/physical/` 或 `src/host/` 中的时序、放置机制、FTL、组合方式或计数逻辑需要遵守最严格的规则，因为项目中的每个结果都依赖它们：

- 说明机制及其证据等级，并为新参数添加来源信息；
- 保持因果关系——任何状态都不能在建模的完成时间之前可见；
- 对非法容量、范围和状态转换采用 Fail-closed；
- 添加一项会在旧行为上失败的聚焦回归测试，并运行 `verify_physical`、`verify_components` 和 `verify_write_amplification`；
- 替换而非累积：在同一次修改中删除过时路径、Flag、测试和文档。

[CONTRIBUTING.md](../../CONTRIBUTING.md) 提供完整检查表，[验证框架](../../verification/README.md)说明各项门禁。进行大型修改前，使用“Feature or model request”模板创建 Issue，是就设计达成一致的最快方式。

## 分享你的发现

完成的研究本身也是贡献。“Share a study, result, or question”Issue 模板会要求填写研究问题、完整命令和结果表（`python3 -m hbfsim show … --format markdown`），便于他人复现和继续研究。
