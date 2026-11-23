# 示例

> 状态：当前版本（Current）  
> 最近审核日期：2026-10-17

这些小型独立脚本分别回答一个有关 HBM + HBF 内存系统的问题。只需先构建模拟器（`python3 -m hbfsim build`），并准备 Python 3.10+，无须其他软件包。脚本可在数秒内运行完毕，并把所有输出写入新的 `out/` 目录。CTest 会运行每个示例，确保它们与模拟器保持同步。

```bash
python3 examples/01_first_comparison.py
```

| 示例 | 问题 | 你将学会 |
| --- | --- | --- |
| [`01_first_comparison.py`](01_first_comparison.py) | HBF 比 HBM 慢多少？混合放置的性能处于什么位置？ | 运行参考场景并从结果中读取数值 |
| [`02_ftl_mapping_tradeoffs.py`](02_ftl_mapping_tradeoffs.py) | 不同 Flash 转换层映射策略在延迟、写放大和磨损方面代价如何？ | 扫描配置 Overlay |
| [`03_read_bandwidth_ceiling.py`](03_read_bandwidth_ceiling.py) | HBF 读取带宽受接口速度等级还是 NAND 阵列限制？ | 使用 `hbfsim.open_session` 直接驱动引擎 |
| [`04_custom_trace.py`](04_custom_trace.py) | 各种放置策略在我自己的访问模式下表现如何？ | 编写带语义标签的 Trace 文件并重放 |
| [`05_custom_policy.py`](05_custom_policy.py) | 我的缓存/晋升方案是否有效？ | 用 Python 事务 DAG 实现放置策略 |
| [`06_capacity_tier_media.py`](06_capacity_tier_media.py) | 作为 HBM 后面的容量层，HBF 与主机 DRAM、CXL 内存、LPDDR 或 SSD 相比如何？ | 比较外部后端设备 |
| [`07_thermal_throttling.py`](07_thermal_throttling.py) | HBF Stack 达到温度上限时会怎样？ | 启用 OCP 热模式（轻度限速、带 CATTRIP 的严重停机）并读取热遥测数据 |
| [`08_fine_trace_frontend.py`](08_fine_trace_frontend.py) | 如何使用明确的前端 Profile 提交细粒度读取？ | 运行 `open_fine_session` 并检查配置/会话 Receipt |

示例 8 也可通过 `python3 -m hbfsim example fine-frontend` 运行；该脚本会调用相同的 CLI 实现。

每个脚本均支持 `--help` 和 `--out DIR`。它们可以直接在新克隆的仓库中运行；执行 `pip install -e .` 后，则可在任意目录中使用 `import hbfsim`。

## 从示例走向正式研究

示例使用小型合成工作负载，因此可在数秒内完成。在得出结论之前：

- 用真实 Trace（示例 4）或 [ServeLoop](../docs/guides/serveloop.md) 生成的 LLM Serving 流量替换合成 Trace；
- 阅读模拟器的全部警告。例如，到达速率超过设备链路能力时，延迟主要反映排队；
- 在 [`configs/parameter-provenance.json`](../configs/parameter-provenance.json) 中核查结论所依赖参数的证据等级。

[入门指南](../docs/getting-started_CN.md)解释这些脚本背后的概念，[扩展 HBFSim](../docs/guides/extending_CN.md)说明如何继续完善示例 5 中原型化的策略。
