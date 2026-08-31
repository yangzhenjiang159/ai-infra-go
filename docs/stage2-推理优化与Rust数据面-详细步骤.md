# 阶段二：推理优化与 Rust 数据面 —— 详细执行手册（第 3-5 个月）

> 目标一句话：**掌握"降本增效"三大武器（量化 / 调度 / 缓存），并交付一个对标 LiteLLM 的 Rust 高性能网关——这是拿 Offer 的核心阶段，不可压缩。**
>
> 前置条件：阶段一已通关。开发机分工：**Mac mini M4 写 Rust 网关**（不需要 GPU），**AutoDL 租卡做量化/压测实验**（用完关机）。

---

## 一、整体路线（12 周两条并行线）

```
周次      理论线（推理优化，每天1-2h）      Rust 线（网关，每天2-3h）
─────────────────────────────────────────────────────────────
第 1 周   量化原理（FP8/FP4/INT4）         Rust 环境搭建 + 所有权
第 2 周   FP8 实操实验（租 4090）          借用/生命周期/结构体
第 3 周   Continuous Batching              Trait/枚举/错误处理
第 4 周   Chunked Prefill + 三层缓存理论   Tokio 异步入门
第 5 周   Speculative Decoding             Axum 入门 + 反向代理练手
第 6 周   （机动/补课）                    网关：非流式代理跑通
第 7 周   ─                                网关：SSE 流式转发
第 8 周   ─                                多模型路由（v1.0 发布）
第 9 周   ─                                语义缓存（v2.0）
第 10 周  ─                                Token 统计 + 限流（v2.0 发布）
第 11 周  ─                                前缀缓存 + 智能降级（v3.0）
第 12 周  必考题串联复盘                    压测 + README + 交付
```

> Rust 进度慢就延后一周进 Part E，理论线可压缩（投机解码只需了解），**不要带着糊涂往上盖楼**。

---

## 二、Part A：量化（第 1-2 周）

### 第 1 周：量化原理（纯看书，不花钱）

| 精度 | 每参数字节 | 7B 权重 | 硬件要求 | 定位 |
|---|---|---|---|---|
| FP16 | 2 字节 | 14 GB | 所有卡 | 基线 |
| **FP8** | 1 字节 | 7 GB | **Hopper(H100)/Ada(4090)** 原生 | **生产默认** |
| **FP4** | 0.5 字节 | 3.5 GB | Blackwell 原生，其余模拟 | 2026 热点 |
| INT4(AWQ/GPTQ) | 0.5 字节 | 3.5 GB | 通用 | 对比项 |

**面试题"为什么倾向 FP8/FP4 而非 INT4"**：
- FP8 是浮点格式，指数动态范围大，量化误差对 LLM 更友好；
- INT4 是定点格式，依赖校准数据集，离线量化流程重；
- H100/4090 的 Tensor Core 原生算 FP8，无转换开销。

> ⚠️ **纠正路线图一个细节（面试加分点）**：A100（Ampere 架构）**没有原生 FP8 Tensor Core**，跑 FP8 是"权重存 FP8、计算转 FP16"的模拟。**原生 FP8 = H100 / RTX 4090（Ada）**。所以 FP8 实验租 **4090（约 2 元/时）就够**，不用租 A100。

**自测**：能口算任意模型三种精度权重；能答"FP8 和 INT4 本质区别"；能答"为什么 4090 能原生跑 FP8 而 A100 不能"。

### 第 2 周：FP8 实操实验（租 4090，预算约 20-30 元）

1. **跑 FP16 基线**（阶段一同款启动命令），压测 `vllm bench serve --num-prompts 500 --request-rate 10`，记录 **TTFT/TPOT/TPS/显存峰值**。
2. **在线 FP8 量化（只加一个参数）**：

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-7B-Instruct \
  --quantization fp8 \
  --gpu-memory-utilization 0.9 \
  --max-num-seqs 256 \
  --port 8000
```

3. 同样压测，记录同样指标，填对比表（权重显存 / TPS / TTFT / 肉眼质量对比）。
4. **关机**。

**交付物**：《FP16 vs FP8 对比报告》（简历"量化对比实验报告"的核心素材）。

---

## 三、Part B：批处理与调度（第 3-5 周，每天 1h 理论）

**第 3 周 Continuous Batching**：静态 batching 为什么浪费（短等长、GPU 空转）→ continuous 如何解决（每步迭代后动态插拔请求）→ 与 PagedAttention 的分工（一个管显存一个管调度）。
自测：能画"5 个请求不同时刻进出 batch"的示意图。

**第 4 周 Chunked Prefill**：长 prompt 的 prefill 阻塞同 batch decode（TPOT 飙升）→ 切块混排解决 → 与 PD 分离的关系（同卡混排 vs 跨集群分离，两个粒度）。
自测：能答"8K prompt 请求进入时，其他请求首字延迟会怎样？两种方案怎么救？"

**第 5 周 Speculative Decoding（了解级）**：小模型 draft 猜 N 个 + 大模型 verify 并行验证；拒绝采样保证输出分布不变；EAGLE-3 在特征层 draft。收益最大场景：接受率高（代码/模板文本）；反亏场景：接受率低。

**交付物**：《调度三大件笔记》（对应必考题 Q1/Q2）。

---

## 四、Part C：三层缓存体系（第 4 周理论，第 9-11 周网关实现）

| 层 | 在哪生效 | 省什么 | 网关模块 |
|---|---|---|---|
| 前缀缓存 | 推理引擎内（vLLM 开 `--enable-prefix-caching` / SGLang 自动） | 相同 system prompt 重复计算 | `cache/prefix.rs` |
| KV 复用 | 引擎内（多租户防语义错位） | 跨请求 KV | `cache/kv_reuse.rs`（理论为主） |
| 语义缓存 | **网关层**（引擎前面） | **整次推理直接跳过** | `cache/semantic.rs`（重点实现） |

**理论要点**：语义缓存 = 请求 → embedding → 向量库余弦检索 → 相似度 > 阈值直接返回；自适应阈值 = 比较"失配成本 vs 推理成本"；**多租户必须 tenant_id 隔离**（否则租户 A 的缓存答租户 B 的问题）。

---

## 五、Part D：Rust 入门（第 1-6 周，Mac 上免费）

> 关键方法：**用 Java 概念锚定 Rust 概念**。

| Java 概念 | Rust 对应 | 关键差异 |
|---|---|---|
| JVM GC 管内存 | 所有权（Ownership） | 编译期决定释放，无 GC |
| 引用（无约束） | 借用（& / &mut） | 同时只能有一个可变借用 |
| null | `Option<T>` | 编译器强制处理空值 |
| Exception | `Result<T, E>` | 错误是返回值，必须处理 |
| interface | trait | 类似但可带默认实现 |
| Maven/pom.xml | Cargo/Cargo.toml | 构建+包管理+测试一体 |
| Netty EventLoop | Tokio runtime | async fn 编译成状态机 |
| CompletableFuture | tokio::spawn | Rust Future 惰性，需 await |

**周计划**：
- 第 1 周：装环境（`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`）+ 官方书 1-4 章（中文版 https://kaisery.github.io/trpl-zh-cn/）+ Rustlings 前三组（https://github.com/rust-lang/rustlings）
- 第 2-3 周：官方书 5-10 章（结构体/枚举/Trait/错误处理），**每学一个概念写一条 Java 对照笔记**
- 第 4 周：Tokio Tutorial 逐章敲完（https://tokio.rs/tokio/tutorial）
- 第 5-6 周：Axum（https://docs.rs/axum/latest/axum/）。练手：①`GET /hello` 返回 JSON；②用 reqwest 写反向代理（请求原样转发到 localhost:8000 再返回）

**第 6 周末自测线**：不看教程能独立写出"接收 POST JSON → 转发上游 → 返回"的 Axum 服务。达不到就延后一周。

---

## 六、Part E：Rust 网关开发（第 6-12 周，`rust-ai-gateway`）

> 先通读 LiteLLM 文档理解"网关该有什么"（https://docs.litellm.ai/）。后端用两种：真实 vLLM（租卡时）+ **mock SSE 服务（免费压测）**。

### 准备：mock SSE 上游（半天，省钱关键）

```python
# mock_llm.py —— 假 vLLM，OpenAI 格式 SSE，每 50ms 吐一个字
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import uvicorn, json, asyncio

app = FastAPI()

@app.post("/v1/chat/completions")
async def chat(req: dict):
    async def gen():
        for i in range(100):
            chunk = {"id": "x", "object": "chat.completion.chunk",
                     "choices": [{"delta": {"content": f"字{i} "}}]}
            yield f"data: {json.dumps(chunk)}\n\n"
            await asyncio.sleep(0.05)
        yield "data: [DONE]\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")

uvicorn.run(app, host="0.0.0.0", port=9000)
```

### v1.0：代理 + 路由 + SSE 转发（第 6-8 周）

目录结构（先聚焦 v1.0 需要的模块）：

```
rust-ai-gateway/
├── Cargo.toml          # axum, tokio, reqwest, serde, tracing, bytes
└── src/
    ├── main.rs         # 启动 Axum
    ├── config.rs       # 路由表：model_name -> upstream_url
    ├── proxy/
    │   ├── request.rs  # 解析 OpenAI 格式请求
    │   ├── sse.rs      # SSE 流式转发（核心难点）
    │   └── response.rs
    └── router/
        └── route.rs    # 按 model 字段选上游
```

**开发顺序（每步可运行再进下一步）**：
1. 非流式代理：解析 body 的 `model` → reqwest 转发 → 原样返回（`stream=false`）。
2. **SSE 流式转发**（核心，预留 3-4 天）：上游 `bytes_stream()` → `axum::response::sse::Sse` 包 `Stream` 透传。要求：**`bytes::Bytes` 零拷贝 + 恒定内存（不攒整个响应）**。
3. 多模型路由：配置文件路由表，未知模型返回 404 JSON。
4. 压测：`wrk -t16 -c1000 -d60s --latency http://localhost:8080/v1/chat/completions`（上游 mock，免费）。

**v1.0 验收**：流式/非流式都能代理；`wrk -c1000` 下内存稳定不涨；改配置加模型不改代码。

### v2.0：语义缓存 + Token 统计 + 限流（第 9-10 周）

1. **语义缓存**：Qdrant 一行起（`docker run -p 6333:6333 qdrant/qdrant`）；embedding 用 Ollama 本地跑（Mac 免费）。流程：请求 → embedding → Qdrant 检索 → 相似度 > 0.95 返回缓存（打 `X-Cache: HIT`）→ 未命中透传并写入。**按 tenant_id 分 collection**。
2. **Token 统计**：流式逐帧解析 `delta.content` 累计（简化：字符数/4；进阶：tiktoken-rs）。每请求打日志：`tenant, model, prompt_tokens, completion_tokens`。
3. **限流**：令牌桶按租户维度（`governor` crate），超限 429 + `Retry-After`。

**v2.0 验收**：相同问题第二次命中缓存（秒级 → 毫秒级）；压测报告有缓存命中率与 Token 节省比；超限 429 不击穿后端。

### v3.0：前缀缓存 + 智能降级（第 11 周）

1. **前缀缓存转发**：识别稳定 system prompt 前缀；vLLM 侧开 `--enable-prefix-caching`；网关侧统计前缀命中率出报表（成本优化的数据素材）。
2. **智能降级**：`tokio::time::timeout` 包住上游首包，>2s 未出首 token → 改路由到备用小模型。

**v3.0 验收**：关掉主模型上游，请求自动落小模型客户端无感；三层缓存全开 vs 全关成本对比报告（目标降 50-60%）。

### 第 12 周：交付冲刺

1. **README**：一句话定位 + 架构图 + 3 条命令快速开始 + 压测数据表 + 与 LiteLLM 对标表。
2. **压测报告**：`benches/sse_bench.rs`（tokio 并发 SSE 客户端）+ wrk 数据成图。
3. **复盘文档**：至少 5 条踩坑（如"SSE 最初用 String 攒 buffer，高并发内存爆，改 Bytes 直通后恒定在 XX MB"——面试 Q8 素材）。

---

## 七、阶段二总验收（对照路线图 3.4）

- [ ] Rust 网关稳定代理流式请求，单机并发 ≥ 1w，内存可控
- [ ] 能讲清 Continuous Batching + Chunked Prefill 如何提升吞吐
- [ ] 能讲清三层缓存各自解决什么、如何叠加，综合成本降 50-60%
- [ ] 能答"并发突增 10 倍怎么不 OOM"（限流 + KV Cache 管理 + 降级）
- [ ] 能答"FP8 vs INT4 怎么选"
- [ ] 能答"你的网关和 LiteLLM 的差异"

---

## 八、参考文章链接

### 量化
- vLLM 官方 FP8 文档：https://docs.vllm.ai/en/latest/quantization/fp8.html
- 知乎《LLM 转 fp8 实践》：https://zhuanlan.zhihu.com/p/2052060740903817314
- CSDN《H100 上 vLLM 运行 FP8 保姆级教程》：https://blog.csdn.net/weixin_30897233/article/details/159306531

### 批处理与调度
- 腾讯云《vLLM 推理加速技术（continuous batching 实现）》：https://cloud.tencent.com/developer/article/2589754
- 腾讯云《Chunked-Prefills 分块预填充机制详解》：https://cloud.tencent.com/developer/article/2540417
- 知乎《Continuous Batching 与 Chunked Prefill：vLLM 调度的核心艺术》：https://zhuanlan.zhihu.com/p/2049247023778490185
- NVIDIA《预测性解码（投机解码）简介》：https://developer.nvidia.com/zh-cn/blog/an-introduction-to-speculative-decoding-for-reducing-latency-in-ai-inference/
- 腾讯云《投机解码原理介绍》：https://cloud.tencent.com/developer/article/2616548

### 语义缓存
- 腾讯云《GPTCache：LLM 应用必备利器》：https://cloud.tencent.com/developer/article/2317992
- GPTCache GitHub：https://github.com/zilliztech/GPTCache

### Rust / Axum / Tokio
- Rust 官方书中文版：https://kaisery.github.io/trpl-zh-cn/
- Rustlings 练习：https://github.com/rust-lang/rustlings
- Tokio 官方教程：https://tokio.rs/tokio/tutorial
- Axum 文档（含 SSE）：https://docs.rs/axum/latest/axum/
- Rust 快速入门（面向 Go+Java 开发者）：https://blog.csdn.net/dudhvd/article/details/161397934
- 腾讯云《Rust 教程知识导图（50 篇系列）》：https://cloud.tencent.com/developer/article/2704048

### 网关对标与压测
- LiteLLM 官方文档：https://docs.litellm.ai/
- CSDN《LiteLLM 企业级 AI 网关架构深度解析》：https://blog.csdn.net/gitblog_00595/article/details/143539710
- wrk 压测工具：https://github.com/wg/wrk
- Qdrant 向量库：https://qdrant.tech/documentation/

---

## 九、省钱与避坑

1. **Rust 开发全程免费**（Mac 本地 + mock 上游 + Ollama embedding + Docker Qdrant），只有量化实验和真机联调租卡。
2. **租卡只做两件事**：第 2 周 FP8 实验、第 11-12 周真实 vLLM 联调（各 1-2 天），其余时间 mock。
3. 用完**立即关机**（阶段一已经付过学费了）。
4. SSE 转发是 v1.0 最大技术风险，**第 7 周如果卡住超过 3 天，把代码发出来求助，别闷头耗**。
5. 每周日晚花 30 分钟写周记（做了什么/卡在哪/下周计划），第 12 周汇总就是复盘文档的骨架。
