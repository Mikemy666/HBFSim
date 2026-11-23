# 文档

> 状态：当前版本（Current）  
> 最近审核日期：2026-10-20

请从顶部开始阅读，获取所需信息后即可停止。“当前版本”表示已实现的行为；“实验性”表示内容可以运行，但其数值尚未经过产品校准。

## 从这里开始

| 文档 | 状态 | 用途 |
| --- | --- | --- |
| [项目概览](../README.md) | 当前版本 | HBFSim 是什么、快速入门及可探索的问题 |
| [入门指南](getting-started_CN.md) | 当前版本 | 从新克隆的仓库开始完成自己的参数扫描 |
| [概念](concepts_CN.md) | 当前版本 | 一页了解 HBF、系统模型、运行流程和术语表 |
| [示例](../examples/README_CN.md) | 当前版本 | 八个可运行脚本，每个回答一个问题 |
| [设计空间实验方法](guides/design-space_CN.md) | 当前版本 | 常见 HBF 设计问题的已验证命令 |
| [扩展 HBFSim](guides/extending_CN.md) | 当前版本 | 配置、工作负载、Python/参考策略和底层机制 |

## 指南

| 文档 | 状态 | 用途 |
| --- | --- | --- |
| [工作负载重放](guides/workload-replay.md) | 当前版本 | 使用参考运行器构建、规范化、重放和检查工作负载 Trace |
| [ServeLoop](guides/serveloop.md) | 当前版本 | 从 LLM Serving 前端驱动引擎 |
| [二进制事务](guides/binary-transactions.md) | 实验性 | 面向大型详细 Trace 的二进制、串行流和实时提交 |
| [Session 内存](guides/session-memory.md) | 实验性 | Session Executor 的事务状态和内存边界 |

## 参考文档

| 文档 | 状态 | 用途 |
| --- | --- | --- |
| [物理模型](reference/model.md) | 当前版本 | 已实现的 HBM、HBF、FTL 和混合机制及其限制 |
| [HBF 热参数](reference/thermal-parameters.md) | 当前版本 | 能耗、RC、边界和阈值的来源及含义 |
| [主机管理的 HBF](reference/host-hbf-management.md) | 当前版本 | Zone、磨损归属、设备 DRAM 和每次运行的磨损报告 |
| [Peer HBM/HBF 驻留](reference/peer-memory.md) | 当前版本 | 原生 Peer 放置、访问新近性、容量压力和后台 KV 写回 |
| [HBF 映射组织](reference/hbf-mapping-organizations.md) | 当前版本 | 映射策略、粒度、写回、内存成本及运行方法 |
| [参考策略套件](reference/scenarios.md) | 当前版本 | 支持的场景 ID、时间计量和解释规则 |
| [可观测性](reference/observability.md) | 当前版本 | Trace/ Summary Schema、磨损报告和运行时防护 |
| [Fine Frontend 基线](reference/fine-frontend.md) | 当前版本 | 固定的 v1 Profile、OCP 派生 Ingress 和软件鉴定 |
| [读取缓冲因果关系](reference/read-buffer-causality.md) | 当前版本 | Fill Handoff、未来容量压力和保留的 Pinning-policy Oracle |
| [证据策略](reference/evidence-policy.md) | 当前版本 | 声明等级、来源信息和未解决的公开标准边界 |
| [配置目录](../configs/README.md) | 当前版本 | 所有 System、Policy Profile 和 Overlay |
| [验证覆盖率](verification/coverage.md) | 当前版本 | 映射到不变量的可执行检查 |
| [项目布局](project-layout.md) | 当前版本 | 代码、数据、测试和文档的位置约定 |

## 目录指南

| 文档 | 状态 | 用途 |
| --- | --- | --- |
| [统一入口](../hbfsim/README.md) | 当前版本 | `python3 -m hbfsim` 命令和 Python API |
| [模拟客户端](../hbfsim_client/README.md) | 当前版本 | 无语义事务协议和持久 Session |
| [源码目录](../src/README.md) | 当前版本 | 生产 C++ 代码的职责边界 |
| [构建目录](../cmake/README.md) | 当前版本 | CMake Target 与测试注册职责 |
| [测试目录](../tests/README.md) | 当前版本 | C++/Python 测试职责及 CTest 分组 |
| [工作负载目录](../workloads/README.md) | 当前版本 | 合成 Trace 生成、局部性分析和工作负载鉴定 |
| [报告目录](../reports/README.md) | 当前版本 | Summary 对比和可视化入口 |
| [验证框架](../verification/README.md) | 当前版本 | Oracle、门禁、Probe、Certificate 和外部证据 |
| [证据目录](../evidence/README.md) | 当前版本 | 校准数值背后的测量和外部模拟器版本固定信息 |
| [外部差分证据](../evidence/external/assets/README.md) | 实验性 | Ramulator2/MQSim 版本、Case 和已验证部分 |
| [DANA A100 Offload 校准](../evidence/hardware/dana_a100_offload/README.md) | 实验性 | `backing/calibrated/dana-*` Overlay 使用的主机 DRAM/NVMe 实测时序 |
| [H200 主机链路测量](../evidence/hardware/host_link/README.md) | 实验性 | `backing/calibrated/h200-*` Overlay 使用的 GPU—主机复制实测时序 |

## 项目与社区

| 文档 | 状态 | 用途 |
| --- | --- | --- |
| [贡献指南](../CONTRIBUTING.md) | 当前版本 | 贡献方式、环境搭建和修改要求 |
| [行为准则](../CODE_OF_CONDUCT.md) | 当前版本 | 社区规范（Contributor Covenant 2.1） |
| [安全策略](../SECURITY.md) | 当前版本 | 支持版本、漏洞报告和不可信输入边界 |
| [变更日志](../CHANGELOG.md) | 当前版本 | 各版本的重要变化 |

## 文档契约

CTest 中的文档门禁 `documentation_contract` 要求每份维护中的文档都出现在本索引中，包含状态和审核日期元数据，所有本地链接均可解析，不含已弃用路径和标识符，并与可执行场景、Schema 和配置契约一致。
