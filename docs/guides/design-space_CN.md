# 设计空间实验方法

> 状态：当前版本（Current）  
> 最近审核日期：2026-10-23

以下每种方法都对应一个常见的 HBF 设计问题，命令已在当前版本上实际运行。结果表取自随附 Profile，仅作示例；方法本身才是重点，数值会随工作负载和参数变化。请先执行 `python3 -m hbfsim build`，再从仓库根目录运行命令。

## 开始前：让工作负载匹配研究问题

内置 Smoke Trace（`--demo-tokens 4 --demo-layers 2`，1,920 次访问；`16`/`8` 约 55,000 次）可在一秒内完成，但只访问几 MiB 数据。它适合观察**延迟、FTL 和写路径**，却无法向容量、链路带宽或热限制施加足够压力。

| 问题取决于…… | 应使用 |
| --- | --- |
| 延迟、映射、写缓冲、WAF | Smoke Trace，必要时扩大为 `--demo-tokens 16 --demo-layers 8` |
| 持续带宽 | 直接向引擎发送 Burst（[示例 3](../../examples/03_read_bandwidth_ceiling.py)）或使用长 Trace |
| GC、磨损和寿命 | 在小型 HBF 上使用写密集 Trace（见下文） |
| 容量和 HBM:HBF 比例 | 通过 [ServeLoop](serveloop.md) 使用真实模型 Footprint |
| 热限速 | 稳态启动（`thermal-steady-state-start`）或持续数秒的负载；见[热限制](#热限制) |

两个习惯可保证扫描公平：比较重放**同一条 Trace** 的参数点（`sweep` 会自动保证），并阅读所有**警告**。Open-loop 到达速率超过设备峰值时，延迟会变成排队时间。如果结论关注延迟，可添加 `--set max-outstanding-requests=64` 实施 Closed-loop 准入。

## HBF 与 HBM 相差多远？哪种放置可以缩小差距？

```bash
python3 -m hbfsim quickstart
```

在 Smoke Trace 上，`all-hbf` 比 `all-hbm` 慢三个数量级；`flat`、`direct-read` 和 `hbf-streaming` 的表现取决于哪些数据保留在 Flash 中。使用 `--set flat-hbm-bytes=...` 改变放置边界，或用 `--set layer-buffer-bytes=...` 改变 Layer Buffer，然后重新运行。[示例 1](../../examples/01_first_comparison.py)展示如何读取具体结果。

## 读取带宽由接口速度等级还是 NAND 感测限制？

```bash
python3 examples/03_read_bandwidth_ceiling.py
```

```text
     grade      tR=4us      tR=2us      tR=1us
    grade1         488         914        1300
    grade2         917        1628        2347
    grade3         918        1631        2355
```

Grade 1 → 2 使带宽翻倍，但 Grade 3 在各种 tR 下都没有收益：每 Stack 256 个 Bank、每 Page 4 KiB 时，阵列并行度和内部路径先于每通道 192 GB/s 的接口达到上限。运行 `run --overlay ocp-v070-grade3` 时，Runner 会直接警告 ECC、原始通道和 TSV 上限低于接口带宽。可同时提高 `hbf-ecc-decode-raw-bw`、`hbf-channel-bw`、`hbf-tsv-bw` 并缩短 `hbf-read-ns`，寻找下一个瓶颈。

## FTL 映射策略的代价是什么？

```bash
python3 -m hbfsim sweep --overlay cached-l2p-1-over-1000 \
  --vary overlay=mapping/page-cache,mapping/entry-cache,mapping/compressed-cache,mapping/block \
  --scenarios all-hbf --demo-tokens 16 --demo-layers 8 --jobs 4 \
  --metrics makespan_us,mean_latency_us,p95_latency_us,hbf_waf,hbf_block_erases
```

```text
overlay                   scenario  makespan (us)  mean lat (us)  p95 lat (us)  HBF WAF  erases
mapping/page-cache        all-hbf           5,441          63.16         594.4    1.792     172
mapping/entry-cache       all-hbf           9,198          24.32         45.81    1.792     172
mapping/compressed-cache  all-hbf           5,700          61.98           492    1.792     172
mapping/block             all-hbf          71,816          521.2         2,947    14.48      13
```

`cached-l2p-1-over-1000` 为 Mapping Cache 分配相当于 HBF 容量 1/1000 的 HBM 预算，扫描的 Overlay 决定 Cache 内容。此处 Entry Cache 以更长 Makespan 换取更低请求延迟；Block Mapping 只需很小的表，却会在小更新时重写整个 Block，导致 8 倍 WAF。完全驻留 Page Table（`mapping/resident`）放不进该预算，扫描会将该参数点报告为拒绝。可用 `--set hbf-ctrl-dram-capacity-denominator=...` 改变预算。各策略见[映射组织参考文档](../reference/hbf-mapping-organizations.md)；[示例 2](../../examples/02_ftl_mapping_tradeoffs.py)是同一研究的脚本版本。

## NAND 编程时间重要吗？

```bash
python3 -m hbfsim sweep --vary hbf-program-ns=37500,75000,150000 \
  --scenarios all-hbf --demo-tokens 16 --demo-layers 8 --jobs 3 \
  --metrics makespan_us,mean_latency_us,p95_latency_us
```

```text
hbf-program-ns  scenario  makespan (us)  mean lat (us)  p95 lat (us)
37500           all-hbf           4,849          25.12         81.03
75000           all-hbf           4,924          25.12         81.03
150000          all-hbf           5,074          25.12         81.03
```

请求延迟没有变化：主机写缓冲先确认写入，再在后台编程 Flash，因此 tPROG 只会延长最后的排空时间（Makespan）。当写入速度超过缓冲能力时，编程时间才进入关键路径；可缩小 `hbf-write-buffer-pages` 或使用写密集 Trace 寻找该临界点。

## 磨损均衡与写放大

研究 GC 和磨损需要足以填满设备的写入。先生成热点写入流，再在缩小的 HBF 上重放（`endurance-regime-scaled` 的原始容量为 4 MiB，因此数秒内就会触发 GC）：

```bash
mkdir -p out
python3 workloads/synthetic/generate.py --output out/hot-writes.trace \
  --pattern hotspot --pages 700 --ops 20000 --read-percent 0 \
  --hot-page-percent 10 --hot-access-percent 90 --seed 1
python3 -m hbfsim sweep --system 4hbm-4hbf --policy none \
  --overlay endurance-regime-scaled --trace out/hot-writes.trace \
  --vary hbf-gc-wear-leveling-weight=0,0.5 --scenarios all-hbf --jobs 2 \
  --metrics makespan_us,hbf_waf,hbf_block_erases,hbf_max_block_erases,hbf_gc_relocations
```

```text
hbf-gc-wear-leveling-weight  scenario  makespan (us)  HBF WAF  erases  max P/E  GC copies
0                            all-hbf       1.876e+06    1.005   2,520      118        106
0.5                          all-hbf       2.401e+06    1.137   2,847       44      2,732
```

没有磨损均衡时，GC 不断复用热点 Block：最旧 Block 达到 118 次擦除，而写入几乎没有放大。权重 0.5 会从新 Block 移出冷数据，把最大值降至 44 次（寿命约提高 2.7 倍），代价是 13% 写放大和 28% 更长运行时间。每个运行目录中的 `summary-hbf-wear/all-hbf.html` 会显示逐 Block 磨损。寿命预测和 Zone 管理见[主机 HBF 管理参考文档](../reference/host-hbf-management.md)。

## HBF 是否比 DRAM、CXL 或 SSD 更适合作为容量层？

```bash
python3 examples/06_capacity_tier_media.py
```

后端 Overlay（`host-dram`、`cxl-memory`、`on-package-lpddr`、`nvme-ssd`、`cxl-ssd`，以及带设备 DRAM Cache 的 `cxl-ssd-cached`）会替换 HBM 后的设备；`external-streaming` 从该设备传输 Layer 的方式与 `hbf-streaming` 从 HBF 传输相同。大多数性能范围是探索性假设；`backing/calibrated/` Overlay 包含实测 A100/H200 主机时序。请留意 Open-loop 警告，或添加 `--set max-outstanding-requests=64`。

## 应配置多少 HBM 和 HBF？

拓扑 Profile `2hbm-6hbf`、`4hbm-4hbf`、`6hbm-2hbf` 和 `eight-stack-baseline` 都保留八个 Stack 槽位，仅改变两者比例（`miniquick/` 变体会缩小容量）。比例问题本质是容量问题，工作负载的 Footprint 必须接近 HBM 容量，例如真实模型和 Batch 的权重加 KV Cache。请通过 [ServeLoop](serveloop.md) 使用 `--system configs/systems/<topology>.cfg` 运行。Smoke Trace 可以全部放入 HBM，因此各种拓扑的结果几乎没有区别。

## 热限制

HBF 遵循 OCP HBF v0.7.0 第 9.2 节定义的四种热模式。每个 Stack 用一个集总 RC 节点表示结温；读取、编程和擦除向节点注入能量，节点向封装边界散热。达到 LTT（随附 Profile 中 `hbf-thermal-throttle-c` 为 85 C）时，固件进入**轻度限速**，按功率预算控制介质操作节奏。`hbf-thermal-throttle-power-w=0` 会自动推导维持 LTT 的预算：边界为 45 C 时每 Stack 18 W，70 C 时为 5.5 W。若仍达到 STT（`hbf-thermal-severe-c`，95 C），则进入**严重限速**：发出 CATTRIP，并在降至 LTT 以下前拒绝命令。达到 `hbf-thermal-shutdown-c`（105 C）时，运行采用 Fail-closed。进入和退出规则见[模型参考文档](../reference/model.md#thermal-reliability-and-evidence-limits)。

随附 Profile 的热时间常数 R×C 为 0.6 s，因此毫秒级 Trace 无法让冷 Stack 明显升温，持续数秒才会接近稳态。`thermal-steady-state-start` 让各 Stack 从 LTT 且已启用限速的状态启动；`thermal-boundary-55c`/`65c`/`70c` 提高封装边界并缩小可持续预算。

`package-coupling-liquid-cooled` 用封装热网络替代固定边界，不能与 `thermal-boundary` Overlay 叠加。700 W GPU 与各 HBM Stack 通过顶盖和中介层热导加热 HBF，边界温度因工作负载而异，网络从空闲稳态启动。再叠加 `package-cascaded-hbf`，每个 HBF Stack 会挂在 HBM 基底裸片后，其 PHY 承担 HBF 流量。在该网络中，Tiered 放置会把每个 HBF 字节写入 HBM 再读出，比直接读取 HBF 的 Peer 放置更容易加热 HBM，进而抬高 HBF 边界温度。

示例 7 分别在关闭热模型、70 C 边界下轻度限速，以及使用该边界无法承受的 16.5 W 预算时发出一次读取 Burst：

```bash
python3 examples/07_thermal_throttling.py
```

```text
               run      budget W          GB/s   paced reads  halted reads       CATTRIP        peak C      light us     severe us
       thermal off             -        1012.1             0             0             0             -           0.0           0.0
  light throttling           5.5         336.1        16,384             0             0          85.0         781.1           0.0
 severe throttling          16.5         453.2        16,384        16,284           652          95.1         318.8         254.5
```

轻度限速把 Burst 限制在功率预算除以每 Bit 能量的水平（每 Stack 5.5 W ÷ 64 pJ/B ≈ 86 GB/s，比阵列低约 3 倍）。16.5 W 预算虽然更快，但 Stack 每隔几微秒压缩时间就升至 STT，CATTRIP 拉高 652 次；Burst 在严重停机中花费 255 µs，在轻度限速中花费 319 µs。冷却系统无法承受的功率预算属于故障条件，而不是性能设置。

每个 Summary 和 Session Receipt 都包含相同的遥测数据：

```bash
python3 -m hbfsim run --system 2hbm-6hbf --overlay thermal-steady-state-start \
  --scenarios all-hbf \
  --metrics makespan_us,hbf_peak_temperature_c,hbf_thermal_paced_ops,hbf_thermal_halted_ops,hbf_cattrip_assertions
```

`summary.json` 的 `hbf_stats.thermal_*` 包含限速/停机操作次数、轻度/严重模式进入次数和持续时间、介质能耗，以及峰值和最终温度；Session 的 `device_delta.hbf` 按 Batch 报告相同计数器，`final_device_totals["hbf"]["state"]` 报告最终状态。能耗和 RC 数值源于公开测量与 Stack 几何结构，边界和阈值则是研究选择，均不是 HBF 产品实测值。[HBF 热参数](../reference/thermal-parameters.md)给出推导与取值范围，因此应比较机制和敏感性，而不是绝对温度。

## 研究你自己的想法

决定哪些数据保留在 HBM 的策略，本质是一个处理事务的 Python 函数。[示例 5](../../examples/05_custom_policy.py)用约 60 行代码实现了晋升 Cache 原型。[扩展 HBFSim](extending_CN.md)说明后续步骤；可使用“Share a study, result, or question”Issue 模板发布研究发现。
