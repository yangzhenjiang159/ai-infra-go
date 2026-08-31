# 阶段四：深水区与简历冲刺 —— 详细执行手册（第 10-12 个月）

> 目标一句话：**交付"别人没有"的差异化项目（PD 分离 KV Cache 传输层），打磨两个王炸项目的叙事，用 4-6 周完成简历与面试冲刺。**
>
> 前置条件：阶段二（Rust 网关）与阶段三（Java 调度平台）均已交付。本阶段硬件策略：**KV 传输层开发全程 Mac 免费 + loopback/局域网压测；只有 PD 真机验证租一次双卡（约 50 元）**。

---

## 一、整体路线（13 周，一条主线 + 冲刺段）

```
周次      内容                                    产出
────────────────────────────────────────────────────────────
第 1 周   多机并行策略：TP/PP/DP/EP 理论          并行策略笔记
第 2 周   MoE 推理原理（DeepSeek 类模型）          MoE 显存/调度笔记
第 3 周   PD 分离架构 + Mooncake/NIXL             PD 分离笔记（必考题 Q5 弹药）
第 4 周   vLLM PD 分离实操（本地 + 租双卡一次）    1P1D 跑通记录
第 5 周   Rust KV 传输代理 v0.1：帧协议打通        单连接 KV 块往返
第 6 周   v0.2：Tokio 多流并发 + 零拷贝           并发压测数据
第 7 周   v0.2 续：局域网压测（Mac↔4070Ti）        P50/P99 延迟 + 带宽数据
第 8 周   v0.3：Block Table 语义 + RDMA 文档级    分块流式 + ACK 确认
第 9 周   README + 压测报告 + 复盘                项目交付
第 10 周  论文精读：FlashAttention + Orca          论文笔记（面试版）
第 11 周  论文精读：ZeRO +（可选）向量库加分项      论文笔记 + 可选 demo
第 12 周  简历打磨：两王炸项目 bullet + 复盘文档    简历定稿
第 13 周  模拟面试 3 轮 + 投递策略执行             开始投递
```

> **优先级铁律**：KV 传输层（第 5-9 周）> 论文精读 > 可选项。若第 5 周前时间已紧张，砍掉第 11 周可选项，绝不动 KV 传输层。

---

## 二、Part A：多机并行策略与 MoE（第 1-2 周，纯理论）

### 第 1 周：TP / PP / DP / EP

| 并行 | 切什么 | 什么时候用 | 通信开销 |
|---|---|---|---|
| **TP**（张量并行） | 单层权重矩阵切多卡 | 单机内 2-8 卡，降延迟 | 每层 2 次 all-reduce，**最贵** |
| **PP**（流水线并行） | 按层切（前 32 层卡 A，后 32 层卡 B） | 跨机（机内带宽不够时） | 只传层边界激活，便宜但有 bubble |
| **DP**（数据并行） | 完整模型复制，请求分流 | 推理扩吞吐的默认姿势 | 无模型通信，只有负载均衡 |
| **EP**（专家并行） | MoE 的专家分散到多卡 | DeepSeek 类 MoE 模型 | all-to-all， routed token 通信 |

**记忆锚点（面试一句话）**：TP 降延迟、PP 破机界、DP 提吞吐、EP 只为 MoE；**机内 TP + 机间 PP + 全局 DP** 是大模型推理的经典组合拳。

**自测**：能答"为什么 TP 不跨机（NVLink 机内 900GB/s vs 跨机 IB 400GB/s，all-reduce 打不起）""TP=8 的 vLLM 为什么需要 gang scheduling"（联动阶段三 Volcano）。

### 第 2 周：MoE 推理特点

- **参数量大、激活少**：DeepSeek-V3 类 671B 总参、每 token 只激活 ~37B → 显存放得下（FP8 约 671GB，仍需多机，但算力需求等效 37B）。
- **调度特点**：负载不均（热门专家被挤爆）→ EP 下要做 expert placement / 重均衡。
- **KV Cache 依然按激活路径产生**，容量压力远小于稠密同参数模型。
- 精度要求：能讲清"MoE 为什么省算力但不省显存""为什么 MoE 部署更适合 PD 分离 + EP 组合"。

---

## 三、Part B：PD 分离与 KV Cache 传输（第 3-4 周理论 + 第 5-9 周项目）

### 第 3 周：PD 分离架构（必考题 Q5 的完整弹药）

1. **回顾阶段一**：Prefill 计算密集（GPU 算力吃满）、Decode 访存密集（带宽瓶颈）→ 混部互相踩踏。
2. **分离收益**：prefill 集群堆算力卡、decode 集群堆带宽卡，各自独立扩缩容、互不干扰；TTFT 和 TPOT 同时优化。
3. **代价与关键问题**：prefill 产生的 **KV Cache 必须跨节点传给 decode**——传多少（GB 级）、怎么传得快（NVLink 出不去，走 RDMA/IB 或高性能 TCP），这就是你要做的 Rust 项目。

4. **业界方案（能报出名字就是加分）**：
   - **Mooncake**（月之暗面，Kimi 背后）：PD 分离 + KV Cache 中心化池（KVCache-centric 调度），论文 https://arxiv.org/abs/2407.00079
   - **NIXL**（Dynamo 生态的高性能传输库）：抽象 UCX/RDMA/NVLink 多种后端
   - vLLM 的 KV Connector 体系：`--kv-transfer-config` 挂载不同 connector

### 第 4 周：vLLM PD 分离实操

1. **本地免费版（4070 Ti，1P1D 单机双实例模拟）**：

```bash
# Prefill 实例（端口 8100）
vllm serve Qwen/Qwen2.5-1.5B-Instruct --port 8100 \
  --kv-transfer-config '{"kv_connector":"SharedStorageConnector","kv_role":"kv_producer","kv_shared_storage_path":"/tmp/kv_store"}'

# Decode 实例（端口 8200）
vllm serve Qwen/Qwen2.5-1.5B-Instruct --port 8200 \
  --kv-transfer-config '{"kv_connector":"SharedStorageConnector","kv_role":"kv_consumer","kv_shared_storage_path":"/tmp/kv_store"}'

# 代理路由（prefill 入口）
vllm serve Qwen/Qwen2.5-1.5B-Instruct --port 8000 \
  --kv-transfer-config '{"kv_connector":"SharedStorageConnector","kv_role":"kv_producer_and_consumer", ...}'
```

> connector 名称以你装的 vLLM 版本文档为准（`vllm.feature_support` 或官方 PD 分离文档），跑不通先查版本——**这个过程本身就是面试素材**（"connector 生态迭代快，我踩过版本兼容坑"）。

2. **真机版（可选，租双 4090 半天，约 50 元）**：两卡各起一个实例，用 `PyNcclConnector`（kv_producer / kv_consumer），观察 KV 传输耗时与 TTFT 变化。
3. **记录**：1P1D vs 单实例混部的 TTFT/TPOT/TPS 对比表（第三个王炸数据点）。

### 第 5-9 周：Rust 项目 `kv-transfer-proxy`（阶段四核心产出）

**一句话定位**：prefill 节点 → decode 节点之间的高性能 KV Cache 传输层（对标 Mooncake transfer layer / NIXL 的极简版），**体现 Rust 性能利刃价值的差异化项目**。

**目录结构**：

```
kv-transfer-proxy/
├── Cargo.toml            # tokio, bytes, serde, criterion, tracing
└── src/
    ├── main.rs           # 启动 proxy（prefill 侧 sender / decode 侧 receiver 两种角色）
    ├── proto.rs          # 帧协议：魔数 + 版本 + 元数据长度 + payload 长度 + body
    ├── meta.rs           # KVMeta { req_id, layer, block_id, num_tokens, dtype, shape }
    ├── sender.rs         # sender：按 block 切片 → 批量发送 → 等 ACK
    ├── receiver.rs       # receiver：收块 → 校验 → ACK → 写入（模拟）目标 buffer
    ├── blocktable.rs     # 模拟 vLLM BlockTable：req_id -> Vec<BlockDesc>（v0.3）
    └── bench/            # criterion 压测：单流/多流延迟、吞吐带宽
```

**版本演进（每步可运行）**：

| 版本 | 内容 | 验收 |
|---|---|---|
| v0.1 | TCP 长连接 + 长度前缀帧，单连接发 1000 个 1MB 块 | 全部到达，校验和一致 |
| v0.2 | Tokio 多流（1 sender → N 条并发连接）、`bytes::Bytes` 零拷贝切分、批量 ACK | loopback 吞吐 ≥ 2GB/s，内存恒定 |
| v0.3 | BlockTable 语义：按 req 流式分块推送 + 完整性 ACK + 模拟"prefill 完成即推"事件 | 端到端模拟 1P1D：模拟 prefill 后 decode 侧可按序读回全部 KV 块 |

**开发要点**：
1. **帧协议先行**（第 5 周第 1-2 天）：`魔数 u32 | 版本 u16 | meta_len u32 | payload_len u64 | meta JSON | payload bytes`。协议定了，sender/receiver 可并行写。
2. **零拷贝是卖点**：`Bytes` 的 `slice`/`copy_to_bytes` 避免 memcpy；对比"逐字节 read 到 Vec"的初版与零拷贝版的压测差异——这就是复盘文档里"优化前后数据"。
3. **局域网真压测（免费）**：Mac 跑 sender，4070 Ti 机器（Linux/WSL2 + Rust）跑 receiver，千兆/两万兆局域网实打实测带宽与 P99。**注意 TCP 粘包/半包处理是必修课**（Java Netty 经验直接迁移，面试可讲"我如何把 Netty 的 LengthFieldBasedFrameDecoder 思想翻译成 Rust 帧解析"）。
4. **RDMA 文档级**（第 8 周，半天）：读 https://github.com/jonhoo/rust-ibverbs README，理解 RDMA 三要素（QP/MR/CQ）与"内核旁路、零拷贝、亚微秒延迟"三个关键词；面试表述："传输层接口我做了 `Transport` trait 抽象，TCP 实现已跑通，RDMA 实现的改造点是 XX"——**讲故事有边界感，不吹**。

**第 9 周交付**：
- README：定位 + 架构图 + 帧协议图 + 压测数据表（loopback + 局域网两栏）。
- 复盘：≥5 条踩坑（粘包、ACK 风暴、Bytes 生命周期、Tokio 背压、延迟毛刺排查）。

---

## 四、Part C：论文精读（第 10-11 周，面试导向）

**方法**：每篇按四步产出笔记，控制在 1 页——① 解决什么问题（痛点+数字）→ ② 核心思想（一句话+一张图）→ ③ 类比你懂的什么（OS/Java 分布式）→ ④ 面试 60 秒讲法。

| 论文 | 一句话核心 | 类比锚点 |
|---|---|---|
| **FlashAttention**（https://arxiv.org/abs/2205.14135） | 感知 SRAM/HBM 的 IO，分块计算 + 重计算换显存访问，精确不近似 | CPU Cache 友好的循环交换优化 |
| **Orca**（OSDI'22，https://www.usenix.org/conference/osdi22/presentation/yu） | iteration-level 调度：每步 decode 后动态插拔请求 = Continuous Batching 鼻祖 | 线程池的 work-stealing：不让槽位空转 |
| **ZeRO**（https://arxiv.org/abs/1910.02054） | 训练侧：优化器状态/梯度/参数三级分片，砍显存冗余 | Redis cluster 的 slot 分片思想 |

> 定位提醒：训练侧只求"能讲清 ZeRO 三级分片是什么、与推理 KV Cache 优化的区别"，不深入。PagedAttention/SGLang 论文阶段一已读，第 13 周面试前串一遍即可。

**第 11 周可选项**（时间富余才做）：HNSW 原理笔记（跳表 + 贪心搜索）或用 Rust 写文档解析 + Chunking + 向量化小 Pipeline（衔接你网关的语义缓存，形成组合故事）。

---

## 五、Part D：简历与面试冲刺（第 12-13 周）

### 第 12 周：简历打磨

**两王炸项目的 bullet 模板（数字必须来自你的真实压测）**：

```
高性能 AI 推理网关（Rust）
- 基于 Tokio/Axum 实现 OpenAI 兼容网关，单机 10w 并发 SSE 长连接，
  峰值内存为 Python 方案 1/10（wrk 实测 XX）
- 设计三层缓存（前缀转发/KV 复用/语义缓存），综合推理成本降低 XX%
- 实现多模型路由与秒级降级，主模型故障切换无感知（客户端零报错）

异构 AI 算力调度平台（Java）
- 基于 Spring Boot 3 + K8s 构建多租户 AI PaaS，GPU 资源池化 + bin-packing
  调度，资源填充率从 XX% 提升至 XX%
- 设计统一异构算力抽象层（NVIDIA/昇腾/海光/摩尔线程），屏蔽设备差异
- 实现 warm pool 弹性伸缩：闲时缩容至 0 释放显存，扩容 P99 首字延迟无毛刺

KV Cache 高速传输层（Rust，差异化项目）
- 为 PD 分离架构实现 prefill→decode KV 块传输代理，帧协议 + 零拷贝 +
  多流并发，局域网实测吞吐 XX GB/s，P99 延迟 XX ms
```

**必做**：把三个项目各写一份 1-2 页复盘文档（背景/架构/踩坑/数据），面试前重读。踩坑记录直接对应路线图 Q8。

### 第 13 周：模拟面试 + 投递

1. **模拟面试 3 轮**（找朋友/AI 面试官都行）：① 系统设计（"设计一个支撑 1w QPS 的 LLM 服务平台"）；② 项目深挖（对着复盘文档互相拷打）；③ 基础串讲（路线图第六节 8 道模拟题全部口头过一遍）。
2. **投递策略**：
   - 岗位关键词：`AI Infra` / `推理平台` / `大模型服务端` / `算力调度` / `GPU 平台` / `LLM Serving`；
   - 渠道：BOSS 直聘 + 猎聘（高级岗）+ 脉脉内推 + 目标公司官网；**优先投：GPU 云厂商、大模型创业公司（做 serving 的）、大厂 AI 平台部门**；
   - 避坑：继续跳过"算法工程师"岗（路线图第七节心法不变）。
3. **谈判锚点**：转型期首份 Offer 以"进圈"优先（40-70 万区间合理），圈内 1-2 年后凭项目跳档（70-120 万），完整逻辑见路线图第九节。

---

## 六、阶段四总验收（对照路线图 5.4）

- [ ] `kv-transfer-proxy` 有可演示原型：1P1D 模拟链路跑通 + 压测数据表
- [ ] 能流畅讲清三个项目各自的"踩坑→定位→解决→数据"（每个 ≥3 条）
- [ ] 3 篇论文各有一页面试版笔记，能 60 秒讲清核心思想
- [ ] 简历定稿：数据全部真实可复现，无一处编造
- [ ] 必考题清单（路线图第六节 Q1-Q8）全部能脱稿作答
- [ ] 完成至少 3 轮模拟面试并复盘口头表达问题

---

## 七、参考文章链接

### 多机并行与 MoE
- HuggingFace《Transformer 并行策略图解》：https://huggingface.co/blog/zh/parallelism
- vLLM 分布式 serving 文档：https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html
- 知乎《MoE 模型的推理优化》：https://zhuanlan.zhihu.com/p/693835861

### PD 分离与 KV 传输
- Mooncake 论文：https://arxiv.org/abs/2407.00079
- vLLM PD 分离文档（Disaggregated Prefill / KV Connector）：https://docs.vllm.ai/en/latest/serving/disagg_prefill.html
- NIXL（NVIDIA Dynamo 传输库）：https://github.com/ai-dynamo/nixl
- rust-ibverbs（RDMA 参考实现）：https://github.com/jonhoo/rust-ibverbs

### 论文
- FlashAttention：https://arxiv.org/abs/2205.14135
- Orca (OSDI'22)：https://www.usenix.org/conference/osdi22/presentation/yu
- ZeRO：https://arxiv.org/abs/1910.02054
- PagedAttention（阶段一已读，复习）：https://arxiv.org/abs/2309.06180

### Rust 进阶
- Tokio 官方文档（channels/backpressure 章节）：https://tokio.rs/tokio/tutorial/channels
- bytes crate（零拷贝核心）：https://docs.rs/bytes/latest/bytes/
- criterion 压测框架：https://github.com/bheisler/criterion.rs

---

## 八、省钱与避坑

1. **总预算 ≈ 50-100 元**：唯一大头是第 4 周租双 4090 验证 1P1D（半天）。KV 传输层开发/压测全部本地（loopback + Mac↔4070Ti 局域网），论文全在 arXiv，简历零成本。
2. **最大坑：想复刻完整 Mooncake**。你做的是"极简传输层 + BlockTable 语义"，2-3 周量级；去追"分布式 KVCache 池 + 调度器"就是无底洞，且面试不要求。
3. **第二坑：论文读太深**。你是工程岗，面试官要"能讲清楚"，不要"能推导公式"。每篇 1 页笔记封顶。
4. **数据造假红线**：简历里每个数字（10w 并发、降本 50%、填充率 85%）都必须能现场用你的压测脚本复现，写"约 XX"也要有底稿。被追问"怎么测的"答不出 = 项目可信度归零。
5. **第 12 周简历没写完就别投**：10 月投出一份平庸简历，11 月有大项目再投同一家公司，不如等一等。冲刺段按"简历定稿 → 模拟面试 → 小公司试水 → 目标公司"的顺序推进，把前几家当练习赛。
