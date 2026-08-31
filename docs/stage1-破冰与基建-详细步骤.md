# 阶段一：破冰与基建 —— 详细执行手册（第 1-2 个月）

> 目标一句话：**从"会调 API"变成"能自己把模型跑起来、压测、讲清楚显存怎么被吃光"。**
>
> 定位：这是后面三个阶段的地基，所有面试"工程体感"素材都从这里积累。
> 时间：6-8 周（原规划 3 个月，因云 GPU 一键镜像 + 成熟文档大幅降低门槛而压缩）。

---

## 一、阶段一整体学习路线（按顺序执行）

```
第 1 步  跑通一次部署（1-2 天）        → 建立信心，确认环境通
第 2 步  理解 Prefill/Decode（3-4 天） → 心法，先懂概念再动手
第 3 步  搞懂显存去哪了（5-7 天）      → 学会手算任意模型显存
第 4 步  理解 PagedAttention（8-11 天）→ vLLM 立身之本，面试必问
第 5 步  部署 + 压测（12-20 天）       → 上云动手，记录三指标
第 6 步  刻意制造 OOM（21-24 天）      → 最重要的"工程体感"
第 7 步  产出交付物（25-28 天）        → 对比笔记 + 面试素材
```

---

## 二、第 1 步：先用最省事的方式跑通一次（第 1-2 天）

不写代码，先建立"我能部署模型"的信心。

1. 打开 AutoDL（首选），租 **A10（24G）** 或 **4090（24G）**，选**自带 vLLM 镜像**的实例。
2. 跟着镜像说明，把 `Qwen2.5-7B-Instruct` 跑起来。
3. 用浏览器打开 OpenAI 兼容接口（默认 `http://localhost:8000`），发一句话，看它回复。

> 目的：**不是学原理，是确认环境通、命令通。**
> 省钱：跑通后立即关机（AutoDL 关机只收存储费，别囤卡）。

**验收**：你能自己复述出"启动一个 vLLM 服务的完整命令"。

---

## 三、第 2 步：理解推理的两个阶段（第 3-4 天，纯看书/文章）

这是整个路线图的心法，**先懂概念再动手**。花两天搞明白两件事：

1. **Prefill（预填充）**
   - 输入 prompt 时，一次性并行算出所有 token 的 KV。
   - 特点：**计算密集**（大量矩阵乘法，GPU 算力吃满）。

2. **Decode（解码）**
   - 一个字一个字往外吐（自回归），每吐一个字要读一次 KV Cache。
   - 特点：**访存密集**（算力利用率低，带宽是瓶颈）。

> **关键结论（面试高频）**：两者资源特征相反，放同一批 GPU 会"互相踩踏"——这正是 **PD 分离（Disaggregated Prefill/Decode）** 的由来，也是 2026 最高频新考点。

**类比记忆**：Prefill 是"一次性读完整道题并算好草稿"，Decode 是"照着草稿一个字一个字写答案"。

**自测**：用一句话给外行讲清楚 Prefill/Decode 的区别。

---

## 四、第 3 步：搞懂"显存去哪了"（第 5-7 天，动手算）

这是你区别于"只会调 API"的关键能力——**学会手算任意模型的显存占用**。

### 4.1 显存三部分

显存 = **权重（Weights）** + **KV Cache** + **激活值（Activations）**

| 项目 | 说明 | 是否可控 |
|---|---|---|
| 权重 | 参数量 × 每个参数字节数 | 由模型和量化精度决定 |
| KV Cache | 随"序列长度 × 并发数"线性增长 | **由运行时动态决定，是 OOM 主因** |
| 激活值 | 依赖 batch 大小 | 由并发决定 |

### 4.2 权重换算（核心，务必亲手推导）

| 精度 | 每参数字节数 | 7B 模型权重 | 70B 模型权重 |
|---|---|---|---|
| FP16 | 2 字节 | 14 GB | 140 GB |
| FP8 | 1 字节 | 7 GB | 70 GB |
| FP4 | 0.5 字节 | 3.5 GB | 35 GB |

> 动手任务：把文档里 "70B 模型 FP16 占 140GB，FP8 占 70GB，FP4 占 35GB" 亲自推导一遍。

### 4.3 KV Cache 粗算（理解它为什么会导致 OOM）

KV Cache 大小取决于：**层数 × 2（K 和 V） × 序列长度 × 隐藏维度 × 字节数**，再乘以**并发请求数**。

> 关键认知：权重是"固定成本"，KV Cache 是"随并发线性上涨的可变成本"。**推理 OOM，几乎都是 KV Cache 撑爆的，不是权重装不下。**

### 4.4 自测

- 不看笔记，能解释"为什么 4070 Ti 12G 只能跑 3B 的 FP16"。
- 能口算 7B 模型 FP16/FP8/FP4 的权重大小。

---

## 五、第 4 步：真正理解 PagedAttention（第 8-11 天）

vLLM 的立身之本，**面试必问**。按"问题 → 解法 → 对比"三步理解。

### 5.1 先理解"问题"

传统推理把 KV Cache 按**固定连续显存块**分配。但不同请求的序列长度不一样，会导致：

- 为最长序列预留显存 → **浪费**；
- 请求长短不一 → **显存碎片化**（就像操作系统的内存碎片化问题）。

### 5.2 再理解"解法"：PagedAttention

vLLM 把 KV Cache 切成**固定大小的小块（block/page）**，像操作系统虚拟内存的"分页"机制一样按需分配：

- 碎片消失（块大小固定，可拼装）；
- 显存利用率大幅提升 → 同样显存能装更多并发请求 → 吞吐提升。

### 5.3 对比：SGLang 的 RadixAttention

RadixAttention 在"分页"基础上，额外用**基数树（Radix Tree）**做**前缀缓存**——相同前缀的 KV 直接复用，跨请求共享。

> 类比记忆：PagedAttention = 内存分页；RadixAttention = 分页 + 前缀去重。
> 这也解释了 vLLM 与 SGLang 的核心差异：vLLM 强在单机低延迟，SGLang 强在前缀复用/多机吞吐。

### 5.4 自测

能回答文档验收标准："PagedAttention 解决碎片化的机制是什么？"

---

## 六、第 5 步：部署 + 压测（第 12-20 天，真正上云动手）

回到 AutoDL，带着目的做，并**记录数据**（面试素材）。

### 6.1 部署 vLLM

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-7B-Instruct \
  --gpu-memory-utilization 0.9 \
  --max-num-seqs 256
```

### 6.2 部署 SGLang（同款模型做对比）

```bash
python -m sglang.launch_server \
  --model-path Qwen/Qwen2.5-7B-Instruct \
  --mem-fraction-static 0.9
```

### 6.3 压测三指标

```bash
# vLLM 自带 bench
vllm bench serve --model Qwen/Qwen2.5-7B-Instruct \
  --num-prompts 1000 --request-rate 10

# SGLang 自带 bench
python -m sglang.bench_serving --num-prompts 1000
```

记录三个指标：

| 指标 | 含义 |
|---|---|
| **TTFT**（首字延迟） | 第一个 token 出现的时间（体现 Prefill 性能） |
| **TPOT**（每 token 延迟） | 后续每个 token 的生成耗时（体现 Decode 性能） |
| **TPS**（吞吐） | 每秒生成的 token 总数 |

### 6.4 对比并思考原因

对比 vLLM vs SGLang 三指标差异，写下猜测（调度器 / 前缀缓存差异）。

> 交付物：一张「vLLM vs SGLang 三指标对比表」，这是简历/笔记里的第一份真实数据。

---

## 七、第 6 步：刻意制造 OOM（第 21-24 天，最重要的"工程体感"）

全阶段**含金量最高**的一步，面试官最爱问"你踩过什么坑"。

1. 用 `nvidia-smi` 实时盯着显存。
2. 逐步调大 `--max-num-seqs` 和 `--gpu-memory-utilization`，观察显存变化，直到 **OOM**。
3. **记录**：OOM 时的 batch 大小、KV Cache 占用、报错信息。

> 为什么重要：文档"面试模拟题 Q8"里最能打动面试官的回答，就是这种"我亲手把显存调爆过，再调回来"的经历。这是你 vs 应届硕博的核心优势，也是简历里最值钱的一段话。

---

## 八、第 7 步：产出交付物（第 25-28 天）

写一份《vLLM vs SGLang 部署对比笔记》，包含：

1. 实测 TTFT/TPOT/TPS 数据；
2. OOM → 调参 → 吞吐变化的记录；
3. 对 Prefill/Decode、PagedAttention、PD 分离的理解。

---

## 九、阶段一验收标准（逐条打勾）

- [ ] 能不看文档独立用 vLLM 和 SGLang 各部署一个模型并压测。
- [ ] 能口头讲清 Prefill/Decode 区别，以及 PD 分离要解决什么问题。
- [ ] 能算清任意模型的显存占用（权重 + KV Cache + 激活值）。
- [ ] 能解释 PagedAttention 解决碎片化的机制。
- [ ] 能回答"调大 batch 后为什么 OOM"。

---

## 十、参考文章链接

### 1. vLLM / PagedAttention

- vLLM 官方文档：https://docs.vllm.ai/
- vLLM 论文《Efficient Memory Management for LLM Serving with PagedAttention》：https://arxiv.org/abs/2309.06180
- 腾讯云《vLLM 核心技术 PagedAttention 原理详解》：https://cloud.tencent.com/developer/article/2529756
- CSDN《vLLM(2)：PagedAttention 论文学习以及原理解析》：https://blog.csdn.net/u013171226/article/details/157032314

### 2. SGLang / RadixAttention

- SGLang 官方文档：https://docs.sglang.ai/
- SGLang 论文《Efficiently Programming Large Language Models using SGLang》：https://arxiv.org/abs/2312.07104
- 腾讯云《原理&图解 vLLM Automatic Prefix Cache（RadixAttention）》：https://cloud.tencent.com/developer/article/2424704
- CSDN《RadixAttention 技术详解：从原理到 SGLang 实践及 vLLM APC 对比》：https://blog.csdn.net/m0_59163425/article/details/159469061

### 3. Prefill / Decode 与 KV Cache

- GitHub《【推理】LLM 推理过程详解》：https://github.com/QingyaFan/blog/issues/201
- 腾讯云《缓存技术：从 CPU Cache 到 AI KV Cache (五) KV Cache》：https://cloud.tencent.com/developer/article/2696934
- 知乎《LLM Engineering 高级 Prefill-Decode 与 KV Cache 调优》：https://zhuanlan.zhihu.com/p/2047442948367856368

### 4. 通用速查

- AutoDL 官网（租卡）：https://www.autodl.com/
- HuggingFace 模型库（Qwen 系列）：https://huggingface.co/Qwen
- vLLM 部署 Qwen 官方教程：https://docs.vllm.ai/en/latest/models/supported_models.html

---

## 十一、学习方法论（后面阶段通用）

面对文档里每阶段的「学习内容」清单，按这个 5 步套路拆：

1. **先定位"为什么"**：这个知识点要解决什么问题？（如 PagedAttention 解决碎片化）
2. **找类比**：它像你熟悉的哪个 Java/操作系统概念？（如分页缓存、bin-packing）
3. **动手验证**：能不能用最小代价跑一下、算一下、调一下？
4. **记录数据**：任何能产出数字的地方，都留下数字（面试素材）。
5. **自测**：对着验收标准打勾，能不看资料复述才算过关。

---

> 阶段一完成后，进入阶段二（Rust 网关）前，先确认所有验收标准已打勾。地基不稳，后面三个阶段的"降本增效"（量化、缓存、网关）都无从谈起。
