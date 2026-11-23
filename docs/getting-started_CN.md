# HBFSim 入门指南（Getting Started）

> 状态：当前版本（Current）  
> 最近审核日期：2026-10-17

本指南将带你从刚克隆的仓库开始，在约 15 分钟内完成自己的参数扫描（Parameter Sweep）。全文统一使用 `python3 -m hbfsim` 命令入口，同时解释每一步在底层执行了什么，避免将实现细节隐藏起来。

## 1. 安装前置环境

HBFSim 需要 **CMake 3.20+、支持 C++20 的编译器以及 Python 3.10+**，不依赖额外的 Python 软件包。

| 平台            | 安装命令                                                     |
| --------------- | ------------------------------------------------------------ |
| Ubuntu / Debian | `sudo apt install build-essential cmake python3 git`（可额外安装 `ninja-build` 以加快构建速度） |
| Fedora          | `sudo dnf install gcc-c++ cmake python3 git`                 |
| macOS           | `xcode-select --install`，以及 `brew install cmake python`   |
| Windows         | 使用安装了 Ubuntu 的 WSL 2，然后按照 Ubuntu 的步骤操作       |

如果不想配置本地环境，可以直接在开发容器（Dev Container）或 GitHub Codespace 中打开仓库。[`.devcontainer/devcontainer.json`](../.devcontainer/devcontainer.json) 会在容器创建时自动构建模拟器。

## 2. 检查环境并构建

```bash
git clone https://github.com/FlashLabAE/HBFSim.git
cd HBFSim
python3 -m hbfsim doctor
python3 -m hbfsim build
```

- `doctor`：检查已具备和缺失的环境组件，并针对每个必需项提供修复建议。
- `build`：在 `build/` 目录中以 Release 模式配置 CMake，并编译两个模拟器程序：`build/hbfsim` 和 `build/hbfsim-reference`。命令入口、示例和 ServeLoop 只需要这两个程序。
- `python3 -m hbfsim build --all`：额外编译测试程序和探针程序（probes）。

如果更习惯直接使用 CMake 或 IDE，也可以执行：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build --parallel
```

其效果相同。仓库还提供 [CMake Presets](../CMakePresets.json)，包括 `release`、`debug`、`sanitize` 以及 `simulator` 构建预设，VS Code 和 CLion 都可以识别。

**所有命令均应在仓库根目录执行。** 如果需要在其他目录使用 `import hbfsim`，可通过可编辑模式安装 Python 包：

```bash
pip install -e .
```

## 3. 运行第一次对比实验

```bash
python3 -m hbfsim quickstart
```

该命令会在 `server-hbm128-hbf512` 系统上重放一段模拟 LLM 行为的合成访存 Trace，包括层权重读取、KV Cache 读取与追加写入，以及临时数据（Scratch）访问。

系统配置为 **4 个 HBM Stack（共 128 GiB）+ 4 个 HBF Stack（共 512 GiB）**。模拟器会比较五种数据放置策略：

- **`all-hbm`**：将所有数据放在 HBM，作为速度最快且不受容量约束的性能上界。
- **`all-hbf`**：将所有数据放到 HBF Flash 转换层（FTL）之后，通过 HBF 的完整逻辑访问路径进行访问。
- **`flat`**：将一个地址空间分区，低地址放在 HBM，其余地址放在 HBF。
- **`direct-read`**：直接从 Flash 读取不可变数据（Immutable Data），将可修改的数据保留在 HBM。
- **`hbf-streaming`**：将权重保存在 HBF 中，并通过双缓冲（Double Buffering）将每层需要的数据传入 HBM。

每个场景的结果表包含以下指标：

| 指标                | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| `makespan`          | 包括后台任务排空（Background Drain）在内，所有工作完全结束所需的时间 |
| `throughput`        | 用户访问的数据字节数 ÷ 最后一条用户请求完成所需的时间        |
| `mean / p95 lat`    | 从请求到达到请求完成的平均延迟 / 第 95 百分位延迟            |
| `HBM ops / HBF ops` | HBM / HBF 实际服务的访问次数，包括后台数据复制               |
| `HBF WAF`           | HBF 写放大系数（Write Amplification Factor）：Flash 实际编程字节数 ÷ 逻辑写入字节数 |

执行以下命令可以查看所有可用统计指标，其中包括擦除次数、磨损最严重的 Block，以及垃圾回收（GC）引起的数据复制次数：

```bash
python3 -m hbfsim list metrics
```

**合理性检查（Sanity Check）：** 运行器会验证每个场景的结果，并输出 `SANITY: PASS` 或一组警告。**出现警告并不意味着模拟器损坏。** 例如，当 Open-loop 请求到达速率超过设备的峰值吞吐能力时，延迟结果可能主要反映排队等待，而不是设备自身的访问延迟；这类结果就需要谨慎解释。

## 4. 查看运行结果

每次运行都会在 `out/` 下创建一个新的结果目录，不会重复使用已有目录，以避免混淆不同实验的输出。

| 文件                | 内容                                                         |
| ------------------- | ------------------------------------------------------------ |
| `command.txt`       | 实际执行的 `hbfsim-reference` 完整命令，可以直接重新运行以复现实验 |
| `resolved.cfg`      | 合并配置文件和命令行覆盖项之后的全部最终配置参数             |
| `summary.json`      | 机器可读的结果，包含来源信息（Provenance）、配置及各场景的统计数据 |
| `stdout.txt`        | 完整的运行报告，包括各阶段的耗时分解                         |
| `trace.txt`         | 本次重放的工作负载 Trace                                     |
| `*-hbf-wear/*.html` | 按 Block 展示的交互式 HBF 磨损分布图，可用浏览器打开         |

`summary.json` 包含每个场景的数百个统计计数器。可以直接用 Python 读取：

```python
import hbfsim

result = hbfsim.load_summary("out/quickstart-YYYYMMDD-HHMMSS")
all_hbf = result.scenario("all-hbf")
print(all_hbf["p95_latency_us"])                 # 主要性能指标
print(all_hbf.raw["hbf_stats"]["page_reads"])   # 访问任意原始计数器
```

如果希望查看请求级时间线（Request-level Timeline），在 `run` 命令中增加：

```bash
--set chrome-trace=out/timeline.json
```

然后使用 `chrome://tracing` 或 [Perfetto](https://ui.perfetto.dev) 打开结果文件。各条时间线 Track 的含义参见[可观测性参考文档](reference/observability.md)。

## 5. 修改一个配置项

HBFSim 采用分层配置机制，按照以下顺序应用，**后面的配置会覆盖前面同名参数的值**：

1. **System（系统配置）**：通过 `--system` 选择一份完整硬件配置。
2. **Reference Policy（参考策略配置）**：若对应系统存在配套的参考策略，则自动选择；使用 `--policy none` 可跳过。
3. **Overlay（叠加配置）**：通过一个或多个 `--overlay` 添加针对单一用途的修改，例如 HBF 速度等级、映射策略或后端存储设备。
4. **Override（单项覆盖）**：通过 `--set KEY=VALUE` 直接修改单个配置项。

示例：

```bash
python3 -m hbfsim list systems
python3 -m hbfsim list overlays
python3 -m hbfsim run --overlay ocp-v070-grade1                # 使用较慢的 HBF 接口
python3 -m hbfsim run --set hbf-read-ns=8000 --scenarios all-hbf  # 使用更慢的 NAND 感测延迟
```

Overlay 和 System 名称都可以简写为没有歧义的文件名，例如 `ocp-v070-grade1`、`mapping/block`。如果名称拼写错误，程序会给出建议。

全部配置项的说明见[配置参数目录](../configs/README.md)。`configs/parameter-provenance.json` 记录了各默认参数的来源。

## 6. 参数空间扫描（Design Space Sweep）

```bash
python3 -m hbfsim sweep \
  --vary hbf-read-ns=2000,4000,8000 \
  --vary overlay=ocp-v070-grade1,ocp-v070-grade2 \
  --scenarios all-hbf --jobs 4
```

每个 `--vary` 会增加一个扫描维度。上面的命令包含：

- HBF 读取延迟：2000、4000、8000 ns（3 种）。
- HBF 接口速度等级 Overlay：Grade 1、Grade 2（2 种）。
- 实验场景：`all-hbf`。
- `--jobs 4`：最多使用 4 个并行作业。

因此，上述命令对应 **3 × 2 = 6 组参数组合**。

扫描过程中所有组合都使用**相同的 Trace**，使得结果差异只来自改变的参数。

扫描轴既可以是任意配置键，也可以是 `system=`、`overlay=` 或 `policy=`；其中 `overlay=none` 表示不添加额外 Overlay。

实验结果输出到：

- `sweep.csv`：每个参数组合及场景占一行，方便用表格工具或 pandas 分析。
- `sweep.json`：机器可读的完整扫描结果。
- 每组参数组合各自的运行目录：保存独立的完整实验输出。

如果某个参数组合不被模拟器接受，程序会报告原因，但仍继续运行其他有效组合。

也可以直接通过 Python API 进行扫描：

```python
points = hbfsim.sweep(vary={"hbf-read-ns": [2000, 4000, 8000]}, scenarios="all-hbf")
for point in points:
    print(point.labels, point.result.scenario("all-hbf")["mean_latency_us"])
```

常见 HBF 问题的经过验证的参数扫描方法见[设计空间实验指南](guides/design-space.md)。

## 7. 导入自己的工作负载

内置 Trace 只是一个 **Smoke Test（冒烟测试）**：它的规模很小，可以快速运行，但**不足以对存储容量或带宽施加有效压力**。要研究真实问题，需要导入自己的工作负载。

### 7.1 自定义 Trace 文件

Trace 文件每行表示一条访存操作，格式如下：

```text
<address> <R|W> [bytes] [kind] [layer=N]
```

其中，`<address>` 为地址，`R|W` 为读写类型，`[bytes]` 为访问字节数，`[kind]` 为数据类别，`[layer=N]` 为可选的 Layer 标记。

相关资源：

- [Example 4](../examples/04_custom_trace.py)：演示如何用 Python 生成自定义 Trace。
- `workloads/synthetic/generate.py`：生成顺序（Sequential）、随机（Random）、跨步（Strided）以及热点（Hotspot）访问模式。
- [工作负载重放指南](guides/workload-replay.md)：介绍如何导入 Trace，以及如何检查其适用性。

使用以下参数重放自己的 Trace：

```bash
--trace FILE
```

### 7.2 LLM Serving 工作负载

[ServeLoop](guides/serveloop.md) 可以根据真实模型描述（Model Descriptors）模拟：

- Continuous Batching（连续批处理）
- Paged KV Cache（分页 KV 缓存）
- Prefix Reuse（前缀复用）

ServeLoop 以闭环（Closed-loop）方式驱动模拟引擎，而不是仅按照预设时间表开放式地发送请求。

## 8. 实现自己的策略（Policy）

模拟引擎能够执行任意事务有向无环图（Transaction DAG）。因此，数据放置、缓存或迁移策略可以通过 Python 代码决定需要发出哪些事务。

[Example 5](../examples/05_custom_policy.py) 通过大约 60 行 Python 代码实现了 HBM Promotion Cache（晋升缓存），并与 `all-hbf`、`all-hbm` 进行了对比。

如果需要更复杂的机制，可以参阅[扩展 HBFSim](guides/extending.md)，了解事务协议，以及当策略复杂到超出简单脚本的能力范围时应该如何扩展模拟器。

## 后续阅读

- [概念](concepts_CN.md)：一页介绍 HBF、系统模型和术语表。
- [示例](../examples/README_CN.md)：八个可以直接运行的示例问题。
- [设计空间实验方法](guides/design-space_CN.md)：针对常见问题的已验证参数扫描命令。
- [扩展 HBFSim](guides/extending_CN.md)：介绍配置、工作负载、策略与底层机制的扩展方法。
- [文档索引](README_CN.md)：全部指南和参考文档的索引。
