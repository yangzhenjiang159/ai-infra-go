# 12 个月 AI Infra 转型路线图（2026 修正版）

> 适用对象：6 年 Java 后端 / 分布式 / 高并发经验，正在向 AI 基础设施方向转型的工程师。
> 核心定位：**Java 控制面（Control Plane）+ Rust 数据面（Data Plane）+ 推理引擎（Engine Layer）** 三层生态位。
> 转型原则：不拼算法/论文，拼系统架构与底层性能；不投"大模型算法工程师"，只投"AI Infra 工程师 / AI 平台架构师 / 大模型后端专家"。

---

## 〇、2026 年相比原规划的关键修正

| 维度 | 原规划认知 | 2026 修正 |
|---|---|---|
| 推理引擎 | vLLM + Triton | **vLLM + SGLang 双雄**，Triton 降级为"了解" |
| 量化 | INT4/INT8（AWQ/GPTQ） | **FP8 生产默认 + FP4 热点**，INT4 作为对比项 |
| 缓存 | 仅语义缓存 | **前缀缓存 / KV 复用 / 语义缓存** 三层体系 |
| 国产算力 | 阶段三"了解" | 升为**主命题**（昇腾/海光/摩尔线程） |
| 推理形态 | 单机单卡 | **PD 分离、多机多卡、MoE** |
| Rust 网关 | 从零自研 | **对标 LiteLLM**（已 Rust 重写内核） |
| 时间 | 12-18 个月 | **压缩为 9-12 个月**，集中火力 |

---

## 〇·五、硬件资源与租赁方案（环境准备）

> 转型全程的硬件分工与租卡策略。核心原则：**本地打基础 + 云 GPU 补短板，按需租、用完关机，别囤卡。**

### 一、现有硬件分工

| 机器 | 用途 | 能否跑推理引擎 |
|---|---|---|
| **RTX 4070 Ti 12G**（已有） | 阶段一学 vLLM/SGLang 原理（用 1.5B-3B 模型） | ✅ 能跑 3B 及以下，7B 需 INT4 量化 |
| **Mac mini M4 16G**（已有） | 阶段二 Rust 网关、阶段三 Java 平台开发 | ❌ 生态不支持 vLLM/SGLang，仅作开发机 |
| **云 GPU（按需租）** | 压测 7B+ 大模型、FP8 量化实验 | ✅ A10/A100 |

### 二、4070 Ti 12G 的显存账（FP16）

- 可用显存 = 12GB − 框架开销（1-2GB）− KV Cache 预留。
- **FP16 最多跑 3B**；7B/8B 需 INT4 量化（权重压到 4-5GB）。
- 无 FP8 原生支持，FP8 实验必须上云 A100/H100。

### 三、显卡租赁平台选型

| 平台 | 定位 | 特点 |
|---|---|---|
| **AutoDL**（首选） | 个人/学生性价比之王 | 卡型全、按小时计费、有 vLLM/SGLang 现成镜像 |
| 恒源云 | 老牌稳定 | 4090/A100 齐全，教程多 |
| 矩池云 | 个人友好 | 按分钟计费，适合短期压测 |
| 智星云 / 揽睿星舟 / 润云 | 新锐性价比 | 4090 类消费卡价格有优势 |

**按需求选卡：**

| 需求 | 卡型 | 参考价格 |
|---|---|---|
| 压测 7B/14B | A10 / 4090（24G） | 2-4 元/时 |
| FP8 量化实验 | A100（40G 起） | 5-10 元/时 |
| PD 分离 / 多机多卡 | 多卡 A100 / H100 | 15-30 元/时 |

**省钱技巧**：用完立即关机释放实例（AutoDL 关机只收存储费）；压测脚本本地写好再上传；模型挂数据盘避免重复下载。

### 四、为什么不租"满血 DeepSeek"（重要认知）

- 满血 DeepSeek（671B MoE）FP8 精度权重约 671GB，需 **16×H800 或 8×H200**，租赁约 **3000-8000 元/天**。
- **完全没必要**：调官方 API 几块钱就能体验；而你要学的 KV Cache / Continuous Batching / PD 分离原理，用 **3B-7B 小模型就能完整学透**，不改变原理。
- **结论**：把钱花在 A10/A100 按小时租小卡做压测（几十元），而非租满血大卡。

---

## 一、总览：四阶段全景

```
┌─────────────────────────────────────────────────────────────┐
│  阶段一（M1-M2）  破冰与基建：把模型跑起来，理解黑盒            │
│  阶段二（M3-M5）  推理优化与 Rust 数据面：让模型跑得省          │
│  阶段三（M6-M9）  Java 控制面与异构调度：让算力跑得稳           │
│  阶段四（M10-M12）深水区与简历冲刺：打造不可替代性              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、阶段一：破冰与基建（第 1-2 个月）

> 目标：脱离"API Caller"身份，掌握本地化部署与推理引擎基础。
> 时间修正：原规划 3 个月 → 压缩至 6-8 周（云 GPU 一键镜像 + 成熟文档大幅降低门槛）。

### 2.1 学习内容

**A. 推理引擎（优先级重排）**
1. **vLLM**：部署 + 理解 PagedAttention（如何解决显存碎片化）。
2. **SGLang**：部署 + 理解 RadixAttention（前缀缓存实现）。
3. **两者差异**（2026 必考）：
   - 单机低延迟 → vLLM；
   - 多机高吞吐 / 复杂调度 → SGLang；
   - 核心差异在调度器设计、前缀缓存、PD 分离连接器生态（vLLM 用 Mooncake/NIXL，SGLang 有自研方案）。
4. Triton Inference Server：**了解即可**（更适合传统 CV/NLP 模型，LLM 场景价值下降）。

**B. 推理两个核心阶段**
- **Prefill（预填充）**：计算密集型，并行处理全部 prompt token。
- **Decode（解码）**：访存密集型，逐 token 自回归生成。
- 关键认知：二者资源特征不同，放同一批 GPU 会"互相踩踏" → 引出 **PD 分离（Disaggregated Prefill/Decode）**，这是 2026 最高频新考点。

**C. 显存与带宽（核心心法）**
- KV Cache 机制 + 为什么推理会 OOM。
- 显存换算：70B 模型 FP16 占 140GB，FP8 占 70GB，FP4 占 35GB。
- PagedAttention / RadixAttention 如何解决碎片化。

**D. 模型格式与转换**
- HuggingFace 格式、Safetensors。
- 导出 ONNX，理解计算图概念。

### 2.2 实操产出

租单卡 GPU（RTX 4090 / A10 / A100），完成：
- 用 vLLM 部署 7B/14B 开源模型（Qwen3 / Llama-4 系列，替代原规划的 Qwen2.5/Llama-3）。
- 用 SGLang 部署同款模型，做对比。
- 压测三指标：**TTFT（首字延迟）、TPOT（每 token 延迟）、TPS（吞吐 tokens/s）**。
- **刻意观察"显存被吃光"**：调 `gpu_memory_utilization`、增大 batch，观察 OOM 与碎片化现象。

### 2.3 可交付物

1. 一份《vLLM vs SGLang 部署对比笔记》，含实测 TTFT/TPOT/TPS 数据。
2. 一段"显存 OOM → 调整参数 → 吞吐变化"的记录（面试"工程体感"素材）。

### 2.4 验收标准

- [ ] 能不看文档独立用 vLLM 和 SGLang 各部署一个模型并压测。
- [ ] 能口头讲清 Prefill/Decode 区别，以及 PD 分离要解决什么问题。
- [ ] 能算清任意模型的显存占用（权重 + KV Cache + 激活值）。
- [ ] 能解释 PagedAttention 解决碎片化的机制。

### 2.5 操作清单：云 GPU 跑 vLLM/SGLang

**Step 1 — 租卡与镜像**
1. AutoDL 租 **A10（24G）** 或 **4090（24G）**，选带 **vLLM** 镜像的实例。
2. 若镜像无 SGLang，`pip install "sglang[all]"`。

**Step 2 — 部署模型（Qwen2.5-7B-Instruct 为例）**

```bash
# vLLM 启动（OpenAI 兼容接口）
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-7B-Instruct \
  --gpu-memory-utilization 0.9 \
  --max-num-seqs 256

# SGLang 启动（同款模型，做对比）
python -m sglang.launch_server \
  --model-path Qwen/Qwen2.5-7B-Instruct \
  --mem-fraction-static 0.9
```

**Step 3 — 压测（用 vLLM 自带 benchmark）**

```bash
# 测吞吐（TPS）
python -m vllm.entrypoints.openai.api_server --model ... &  # 已启动则跳过
vllm bench serve --model Qwen/Qwen2.5-7B-Instruct \
  --num-prompts 1000 --request-rate 10

# 也可用 SGLang 的 bench 脚本
python -m sglang.bench_serving --num-prompts 1000
```

**Step 4 — 刻意观察"显存被吃光"**
- 用 `nvidia-smi` 观察显存变化。
- 逐步调大 `--max-num-seqs` / `--gpu-memory-utilization`，直到 OOM。
- 记录：OOM 时的 batch 大小、KV Cache 占用、报错信息。

**Step 5 — 记录三指标（面试素材）**
- **TTFT**（首字延迟）：第一个 token 出现的时间。
- **TPOT**（每 token 延迟）：后续每个 token 的生成耗时。
- **TPS**（吞吐）：每秒生成的 token 总数。
- 对比 vLLM vs SGLang 三指标差异，并思考原因（调度器/前缀缓存差异）。

**验收要点**：能独立跑通上述流程，且能回答"调大 batch 后为什么 OOM"。

---

## 三、阶段二：推理优化与 Rust 数据面（第 3-5 个月）

> 目标：掌握企业级最看重的"降本增效"，并启动 Rust 高性能网关项目。
> 时间修正：这是**投入产出比最高**的阶段，是拿 Offer 的核心抓手，不可压缩。

### 3.1 学习内容

**A. 量化（技术路线换血）**
1. **FP8**（生产默认精度）→ 理解权重/激活联合量化，硬件原生支持。
2. **FP4**（2026 热点）→ 4-bit 浮点，vLLM/SGLang 都在推。
3. INT4（AWQ/GPTQ/SmoothQuant）→ **作为对比项**，能回答"为什么倾向 FP8/FP4 而非 INT4"。
4. 实操：FP16 → FP8 量化，对比精度损失 / 显存 / 速度变化。

**B. 批处理与调度**
- Continuous Batching（连续批处理）。
- **Chunked Prefill**：理解它如何解决"长 prompt 阻塞短请求"。
- Speculative Decoding（投机解码）：小模型 draft + 大模型 verify 的批量验证；了解 EAGLE-3。

**C. 三层缓存体系（2026 降本核心框架）**
1. **前缀缓存（Prefix Caching）**：跨请求共享相同前缀的 KV Cache，最省成本（省 50%+）。
2. **KV Cache 复用 / 语义对齐**：多租户、灰度、A/B 场景下的语义错位风险（tenant_id、schema_version 维度编码）。
3. **语义缓存（Semantic Cache）**：向量相似度 + 自适应"失配成本 vs 推理成本"决策。

**D. Rust 发力点 1：高性能 AI 代理网关（对标 LiteLLM）**

- 技术栈：Rust + Tokio + Axum/Hyper + Redis。
- **对标而非造轮子**：LiteLLM 已用 Rust 重写内核（"Rust core with Python SDK"），Cloudflare AI Gateway / Portkey 均为高性能栈。
- 核心模块：
  1. 多模型路由 + 智能降级（大模型排队超时 → 路由小模型）。
  2. SSE 流式响应高并发代理（Tokio 异步非阻塞）。
  3. Token 消耗精准统计。
  4. 语义缓存拦截（向量相似度命中直接返回）。
  5. 前缀缓存转发（对接 vLLM/SGLang 的前缀缓存接口）。
- 核心卖点：单机支撑 **10w+ 并发 SSE 长连接，内存仅为 Java/Python 版 1/10**。

### 3.2 实操产出

1. Rust 网关 v1.0：能代理 OpenAI 格式 API，支持多模型路由 + 流式转发。
2. Rust 网关 v2.0：接入语义缓存 + Token 统计。
3. 量化对比实验报告。

### 3.3 可交付物

1. GitHub 上的 Rust 网关项目（含 README、压测报告、架构图）。
2. 一份量化对比报告（FP16/FP8/INT4 的精度-显存-速度矩阵）。

### 3.4 验收标准

- [ ] Rust 网关能稳定代理流式请求，压测单机并发 ≥ 1w，内存可控。
- [ ] 能讲清 Continuous Batching + Chunked Prefill 如何提升吞吐。
- [ ] 能讲清三层缓存各自解决什么、如何叠加，综合成本下降 50-60%。
- [ ] 能回答"并发突增 10 倍怎么不 OOM"（限流 + KV Cache 管理 + 降级）。

### 3.5 Rust 网关技术方案

**项目名**：`rust-ai-gateway`（对标 LiteLLM）

**技术栈**：Rust + Tokio + Axum + Redis（缓存）+ Qdrant/向量库（语义缓存）

**目录结构**：

```
rust-ai-gateway/
├── Cargo.toml
├── src/
│   ├── main.rs                 # 入口，启动 Axum 服务
│   ├── config.rs               # 配置（模型路由表、限流阈值）
│   ├── router/                 # 多模型路由
│   │   ├── mod.rs
│   │   ├── route.rs            # 路由决策（含智能降级）
│   │   └── fallback.rs         # 大模型排队超时 → 小模型降级
│   ├── proxy/                  # 核心代理层
│   │   ├── mod.rs
│   │   ├── sse.rs              # SSE 流式响应高并发转发
│   │   ├── request.rs          # OpenAI 格式请求解析
│   │   └── response.rs         # 流式/非流式响应封装
│   ├── cache/                  # 三层缓存
│   │   ├── mod.rs
│   │   ├── prefix.rs           # 前缀缓存（对接 vLLM/SGLang）
│   │   ├── kv_reuse.rs         # KV Cache 复用/语义对齐
│   │   └── semantic.rs         # 语义缓存（向量相似度）
│   ├── billing/                # Token 统计
│   │   ├── mod.rs
│   │   └── token_counter.rs    # 流式 token 精准计量
│   ├── limit/                  # 限流降级
│   │   ├── mod.rs
│   │   └── rate_limit.rs       # 令牌桶/滑动窗口
│   └── metrics/                # 可观测性
│       └── mod.rs              # Prometheus 指标暴露
└── benches/
    └── sse_bench.rs            # 并发压测
```

**核心模块实现要点**：

1. **SSE 高并发转发**（`proxy/sse.rs`）
   - 用 `tokio::spawn` 处理每个连接，避免阻塞。
   - 上游响应流直接透传，用 `bytes::Bytes` 零拷贝。
   - 关键：`axum::response::sse::Sse` 流式返回，内存恒定。

2. **多模型路由 + 降级**（`router/`）
   - 路由表：`model_name -> upstream_url + 优先级`。
   - 降级逻辑：主模型排队时长 > 阈值（如 2s）→ 低复杂度 query 路由到 7B 小模型。

3. **语义缓存**（`cache/semantic.rs`）
   - 请求 → embedding（调 embedding 模型）→ Qdrant 余弦相似度检索。
   - 相似度 > 阈值 → 直接返回缓存响应，跳过推理。
   - 自适应：比较"缓存失配成本"与"推理成本"，动态调阈值。

4. **前缀缓存转发**（`cache/prefix.rs`）
   - 识别请求中稳定的 system prompt 前缀。
   - 对接 vLLM（`--enable-prefix-caching`）/ SGLang（RadixAttention 自动启用）。

5. **Token 精准统计**（`billing/token_counter.rs`）
   - 流式场景按 `data:` 帧增量累计 token。
   - 对接 tokenizer 或上游返回的 `usage` 字段。

6. **限流降级**（`limit/rate_limit.rs`）
   - 令牌桶按租户维度限流。
   - 超限时：降级到小模型 或 返回 429 + 排队提示。

**分阶段开发计划**：

| 版本 | 功能 | 验收 |
|---|---|---|
| v1.0 | 代理 OpenAI 格式 API + 多模型路由 + SSE 转发 | 单机并发 1w+ |
| v2.0 | 语义缓存 + Token 统计 + 限流 | Token 成本降 35%+ |
| v3.0 | 前缀缓存转发 + 智能降级 | 综合成本降 50-60% |

**压测方法**：
```bash
# 用 wrk 或 vegeta 打 SSE 长连接
wrk -t 16 -c 10000 -d 60s --latency http://localhost:8080/v1/chat/completions
# 或用自定义 Rust bench（benches/sse_bench.rs）模拟流式请求
```

---

## 四、阶段三：Java 控制面与异构调度（第 6-9 个月）

> 目标：发挥 Java 架构优势，构建企业级 AI PaaS 平台，向"架构师"迈进。
> 时间修正：异构算力调度从"加分项"升为**主命题**，是本阶段重心。
> 📄 详细执行手册：[stage3-Java控制面与异构调度-详细步骤.md](./stage3-Java控制面与异构调度-详细步骤.md)

### 4.1 学习内容

**A. GPU 资源池化与 K8s 调度**
- K8s GPU 调度机制（Device Plugin）。
- MIG（多实例 GPU）、vGPU 技术：显存/算力的切片共享。

**B. 异构算力适配（2026 大厂刚需，主命题）**
- 统一抽象 NVIDIA GPU + 华为昇腾（CANN/MindIE）+ 海光 DCU（ROCm 兼容）+ 摩尔线程。
- 设计抽象算力调度层，屏蔽底层硬件差异。
- 了解各家的 Device Plugin 与调度差异。

**C. 全链路监控与 AIOps**
- Prometheus + Grafana。
- 关键指标（远超 CPU/内存）：GPU 利用率、显存带宽、**KV Cache 命中率**、TTFT/TPOT 的 P99。
- 追踪链路：API 网关 → 推理引擎 → KV Cache。

**D. 弹性伸缩（Serverless AI）**
- 闲时卸载非核心模型显存（缩容至 0）。
- 请求到来时预热冷启动（3 秒内）。

### 4.2 实操产出（简历核心项目）

**项目：企业级异构 AI 算力调度与 MLOps 平台**
- 技术栈：Java（Spring Boot/Cloud）+ K8s + Prometheus + Python（引擎侧胶水）。
- 功能模块：
  1. 模型版本管理 + 一键部署。
  2. 异构算力资源池化调度器（GPU 显存级隔离与 vGPU 共享）。
  3. Serverless 弹性伸缩（缩容至 0 + 预热冷启动）。
  4. 多租户配额管理。
  5. 全链路监控大盘。

### 4.3 可交付物

1. Java 平台核心代码（GitHub 可展示）。
2. 一份《异构算力调度层设计文档》（抽象资源模型 + 各厂商适配方案）。
3. 监控大盘截图 + 一次压测报告。

### 4.4 验收标准

- [ ] 平台能完成模型一键部署 + 多租户隔离 + 弹性伸缩。
- [ ] 能画出"API 网关 → 推理引擎 → KV Cache"全链路监控拓扑。
- [ ] 能讲清 NVIDIA / 昇腾 / 海光 / 摩尔线程的调度差异与统一抽象思路。
- [ ] 能回答"如何把 GPU 利用率从 30% 提升到 75%"。

### 4.5 Java 异构算力调度平台技术方案

**项目名**：`ai-orchestrator`（企业级 AI 算力调度与 MLOps 平台）

**技术栈**：Java 17 + Spring Boot 3 + Spring Cloud + K8s + Prometheus + Python（引擎侧胶水）

**目录结构**：

```
ai-orchestrator/
├── pom.xml
├── orchestrator-core/          # 核心调度模块
│   └── src/main/java/com/ai/
│       ├── scheduler/          # 调度器
│       │   ├── GpuResourcePool.java   # GPU 资源池抽象
│       │   ├── PlacementStrategy.java # 放置策略（bin-packing）
│       │   └── vgpu/                  # vGPU 切片
│       │       └── VgpuAllocator.java
│       ├── heterogeneous/      # 异构算力适配层
│       │   ├── GpuProvider.java       # 统一抽象接口
│       │   ├── NvidiaProvider.java    # NVIDIA 实现
│       │   ├── AscendProvider.java    # 昇腾（CANN）实现
│       │   ├── HygonProvider.java     # 海光 DCU 实现
│       │   └── MooreThreadsProvider.java # 摩尔线程实现
│       ├── model/              # 模型管理
│       │   ├── ModelRegistry.java     # 模型版本管理
│       │   └── ModelDeployer.java     # 一键部署
│       ├── autoscale/          # 弹性伸缩
│       │   ├── ScaleController.java   # 缩容至 0 + 预热
│       │   └── WarmupManager.java     # 冷启动预热
│       ├── tenant/             # 多租户
│       │   └── QuotaManager.java      # 配额管理
│       └── monitor/            # 监控
│           └── MetricsCollector.java  # 接入 Prometheus
├── orchestrator-api/           # REST API 层
│   └── ...（Controller + DTO）
└── deploy/
    ├── k8s/                    # K8s 部署清单
    │   ├── device-plugin.yaml
    │   └── scheduler-config.yaml
    └── docker/Dockerfile
```

**核心设计要点**：

1. **异构算力抽象层**（`heterogeneous/`）
   - 定义 `GpuProvider` 统一接口：`allocate() / release() / getMetrics() / getCapability()`。
   - 各厂商实现屏蔽差异：
     - NVIDIA：走 K8s Device Plugin + CUDA。
     - 昇腾：CANN + MindIE，Device Plugin 用昇腾插件。
     - 海光 DCU：ROCm 兼容层。
     - 摩尔线程：MUSA 生态。
   - 这是 2026 招聘**主命题**，是简历最大亮点。

2. **GPU 资源池化调度**（`scheduler/`）
   - 资源模型：显存（GB）+ 算力（TFLOPS）+ 带宽三个维度。
   - `PlacementStrategy`：bin-packing 算法，减少碎片，提升利用率。
   - `VgpuAllocator`：显存级隔离 + MIG/vGPU 切片共享。

3. **弹性伸缩（Serverless AI）**（`autoscale/`）
   - `ScaleController`：闲时卸载非核心模型显存（缩容至 0）。
   - `WarmupManager`：请求到来时预热，3 秒内冷启动。
   - 关键：预热用模型小样本 + 预加载权重到显存。

4. **多租户配额**（`tenant/`）
   - `QuotaManager`：按租户分配 GPU 显存/算力/并发额度。
   - 超额时降级或排队。

5. **全链路监控**（`monitor/`）
   - 采集 GPU 利用率、显存带宽、KV Cache 命中率、TTFT/TPOT P99。
   - 链路：API 网关 → 推理引擎 → KV Cache。

**分阶段开发计划**：

| 版本 | 功能 | 验收 |
|---|---|---|
| MVP | 模型一键部署 + 多租户配额 | 能部署 vLLM 模型 + 隔离租户 |
| v1.0 | GPU 资源池化 + vGPU 切片 | GPU 利用率 30% → 60%+ |
| v2.0 | 异构适配（NVIDIA + 昇腾）+ 弹性伸缩 | 冷启动 ≤ 3s，缩容至 0 |

---

## 五、阶段四：深水区与简历冲刺（第 10-12 个月）

> 目标：补齐差异化亮点，冲刺面试。
> 时间修正：向量数据库底层**降级为可选加分项**，核心护城河仍是"Java 分布式 + Rust 高性能 + 推理引擎"三件套。
> 📄 详细执行手册：[stage4-深水区与简历冲刺-详细步骤.md](./stage4-深水区与简历冲刺-详细步骤.md)

### 5.1 学习内容（按优先级）

1. **【必做】Rust 发力点 3：PD 分离 KV Cache 传输代理**
   - prefill 集群与 decode 集群间通过高速网络传 KV Cache。
   - 用 Rust（Tokio + RDMA/IB）写低延迟 KV Cache 传输层。
   - 这是"别人没有、含金量极高"的差异化项目，最能体现 Rust 性能利刃价值。

2. **【必做】经典论文精读**
   - FlashAttention（IO 感知）。
   - ZeRO（分布式训练显存优化）。
   - Orca（推理调度）。

3. **【可选】向量数据库底层**
   - Milvus / Qdrant 底层（HNSW、DiskANN）。
   - 用 Rust 写文档解析 + 向量化 Pipeline（PDF/Word 并发解析、清洗、Chunking、高吞吐写入）。

4. **【必做】多机并行策略**
   - TP/PP/DP/EP（专家并行）四类并行的适用场景与通信开销。
   - MoE 推理（DeepSeek 类模型）的显存与调度特点。

### 5.2 简历项目打磨

- 两个"王炸"项目：**Rust 高性能网关** + **Java 异构算力调度平台**。
- 数据要有冲击力（可基于真实压测）：
  - 网关：10w+ 并发 SSE，内存 1/10，Token 成本降 50-60%。
  - 平台：GPU 利用率 30% → 75%，冷启动 ≤ 3 秒。

### 5.3 可交付物

1. 更新后的简历 + 两个项目的深度复盘文档（含踩坑记录）。
2. 一篇技术博客（可选，提升曝光）。

### 5.4 验收标准

- [ ] PD 分离 KV 传输层有可演示的原型。
- [ ] 能流畅讲清每个项目"踩过的坑"（工程体感，是区别于应届硕博的核心优势）。
- [ ] 能完整回答 3 个必考题（见下）。

---

## 六、必考面试题清单（贯穿全程）

1. **"并发突增 10 倍，推理服务怎么表现？如何保证不 OOM？"**
   → Continuous Batching、KV Cache 管理、限流降级、PD 分离。

2. **"如何降低大模型推理延迟和成本？"**
   → 量化（FP8/FP4）、Speculative Decoding、三层缓存（前缀/复用/语义）。

3. **"Java / Rust / Python 在你的架构中如何协同？"**
   → 控制面（Java）/ 数据面（Rust）/ 引擎胶水（Python）的边界划分。

4. **"vLLM 和 SGLang 怎么选？"**（2026 新增高频）
   → 单机延迟选 vLLM，多机吞吐/复杂调度选 SGLang。

5. **"P/D 分离怎么实现？KV Cache 怎么跨机传输？"**（2026 新增高频）
   → 计算/访存分离，Mooncake/NIXL 或自研 Rust 传输层。

### 6.1 面试模拟题库（附参考答题要点）

**Q1. 并发突增 10 倍，你的推理服务会怎么表现？如何保证不 OOM？**

> 参考要点：
> - 现象：KV Cache 显存被快速占满 → 触发 OOM 或排队延迟激增。
> - 应对：
>   1. Continuous Batching 提升吞吐，让显存利用率更高。
>   2. KV Cache 管理：限制 `max_num_seqs`、开启 prefix caching 复用。
>   3. 限流：网关层令牌桶按租户限流。
>   4. 降级：高复杂度请求路由到小模型，或排队。
>   5. 终极方案：PD 分离，prefill/decode 分集群，避免互相踩踏。

**Q2. 如何降低大模型推理延迟和成本？**

> 参考要点：
> - 量化：FP8（生产默认）/ FP4（热点），显存减半，速度提升。
> - Speculative Decoding：小模型 draft + 大模型 verify。
> - 三层缓存：前缀缓存（省 50%+）/ KV 复用 / 语义缓存。
> - 调度：Continuous Batching + Chunked Prefill。
> - 硬件：异构算力选型（国产卡成本更低）。

**Q3. Java / Rust / Python 在你的架构中如何协同？**

> 参考要点：
> - 控制面（Java）：AI PaaS 平台、GPU 调度、多租户、编排、网关，发挥分布式系统经验。
> - 数据面（Rust）：高性能推理网关、SSE 代理、KV 传输层，极致性能。
> - 引擎胶水（Python）：vLLM/SGLang 配置、模型导出、压测脚本。
> - 边界：Java 管"调度和平台"，Rust 管"性能和连接"，Python 管"引擎"。

**Q4. vLLM 和 SGLang 怎么选？**

> 参考要点：
> - 单机低延迟 → vLLM（PagedAttention 成熟）。
> - 多机高吞吐 / 复杂调度 → SGLang（RadixAttention 前缀缓存更强）。
> - 差异：调度器设计、前缀缓存实现、PD 分离连接器生态。

**Q5. P/D 分离怎么实现？KV Cache 怎么跨机传输？**

> 参考要点：
> - Prefill（计算密集）与 Decode（访存密集）分集群部署。
> - KV Cache 通过高速网络传输（vLLM 用 Mooncake/NIXL）。
> - 自己可用 Rust + RDMA/IB 写低延迟传输层。
> - 收益：prefill 集群专注算力，decode 集群专注访存，互不干扰。

**Q6. 你的 Rust 网关相比 LiteLLM 有什么优势？**

> 参考要点：
> - 对标而非造轮子：LiteLLM 已 Rust 重写内核，说明 Rust 是正确技术选型。
> - 差异化：自研语义缓存 + 前缀缓存三层体系，Token 成本降 50-60%。
> - 性能：单机 10w+ 并发 SSE，内存 1/10。

**Q7. 如何把 GPU 利用率从 30% 提升到 75%？**

> 参考要点：
> - 资源池化 + bin-packing 放置策略，减少碎片。
> - vGPU/MIG 切片共享，显存级隔离。
> - 弹性伸缩：闲时缩容至 0，忙时预热冷启动。
> - 多租户配额，提升整体填充率。

**Q8.（追问）你踩过什么坑？**

> 参考要点（工程体感，务必真实）：
> - "压测 vLLM 时显存碎片化严重，调 `gpu_memory_utilization` + 开 PagedAttention，吞吐提升 X 倍。"
> - "FP8 量化在 4070 Ti 上是软件模拟，数据没参考价值，得换 A100。"
> - "SSE 长连接用同步线程池会占满内存，改用 Tokio 异步后内存降了 10 倍。"

---

## 七、避坑指南（普通本科 + 30 岁破局心法）

1. **不投**："大模型算法工程师""AI 研究员"（卡学历、卡论文）。
2. **死盯**："AI Infra 工程师""AI 平台架构师""大模型后端专家"。
3. **扬长避短**：多谈工程体感，少谈公式推导。
   - 示例："压测 vLLM 时发现显存碎片化严重，调 `gpu_memory_utilization` + 开 PagedAttention，吞吐提升 X 倍。"
4. **Rust 是加分项不是必选项**：面试官不懂 Rust 时，强调其内存安全、极低延迟、高并发连接数的架构收益，作为"追求极致性能"的技术品味证明。

---

## 八、里程碑检查表

| 时间 | 里程碑 | 状态 |
|---|---|---|
| M2 末 | vLLM + SGLang 部署对比完成 | ☐ |
| M5 末 | Rust 网关 v2.0（含语义缓存）上线 | ☐ |
| M9 末 | Java 异构调度平台 MVP 完成 | ☐ |
| M12 末 | PD 分离 KV 传输原型 + 简历冲刺 | ☐ |

---

## 九、薪资预期（2026 年一线城市参考）

> 注意区分"工程岗"与"算法岗"：你的目标岗位是**工程岗**（AI Infra / 大模型后端 / AI 平台架构），不卡学历不卡论文，与"大模型算法研究员"是两回事。

| 岗位 | 中级(2-5年) | 高级/专家(5年+) |
|---|---|---|
| AI Infra 工程师 | 45-70K/月 | 70-100K+/月 |
| 大模型后端 / 推理优化 | 40-60K/月 | 60-90K/月 |
| AI 平台架构师 | 45-70K/月 | 70-120K+/月 |

换算年薪（14-16 薪）：

- **中级 AI Infra / 大模型后端**：**55-90 万**
- **高级/专家 AI Infra / AI 平台架构师**：**90-150 万+**

**结合你的情况（6 年 Java + 转型中）：**

1. **转型初期**（学完 + 1-2 个项目）：以"资深后端 + AI Infra 项目经验"进入，**40-70 万**是合理区间，已比传统 Java 后端（约 30K/月）有明显溢价。
2. **转型成功**（2-3 个硬核项目落地）：冲刺 **AI Infra 高级 / AI 平台架构师**，**70-120 万**是现实目标。

**拉高薪资的关键变量**：项目真实落地 > 只会调 API；能讲清 KV Cache OOM、量化调优等工程体感 > 背概念；有 Rust 真实性能数据 > 不会 Rust；懂昇腾/海光异构调度 > 只懂 NVIDIA。

---

> 总结：AI Infra 是"越老越妖"的赛道，需要懂底层硬件、分布式系统、网络通信，正是 6 年 Java 后端经验的完美延伸。用 Java 构建宏大调度平台，用 Rust 打磨极致性能刀刃，按本路线图执行 12 个月，完全有能力在 32-33 岁拿到 AI 基础设施领域的高级/专家 Offer。
