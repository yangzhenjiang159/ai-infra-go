# 阶段三：Java 控制面与异构调度 —— 详细执行手册（第 6-9 个月）

> 目标一句话：**用 6 年 Java 功底构建"企业级异构 AI 算力调度与 MLOps 平台"（`ai-orchestrator`）——这是你的主场阶段，也是简历第二个王炸项目。**
>
> 前置条件：阶段二已通关（Rust 网关 v3.0 交付）。开发机分工：**Mac mini M4 写 Java 平台 + 本地 K8s（全程免费）**，GPU 相关实验尽量用 mock/时间片方案，**只有必要时才租卡**。

---

## 一、整体路线（18 周，两条并行线）

```
周次      K8s/云原生线（理论+动手，每天1h）    Java 平台线（ai-orchestrator，每天2-3h）
──────────────────────────────────────────────────────────────────────
第 1 周   K8s 核心对象快速过（你有分布式底子）  ─
第 2 周   GPU 上 K8s：Device Plugin 原理       ─
第 3 周   本地 kind 集群 + fake-gpu-operator   项目骨架（Spring Boot 3 + 模块划分）
第 4 周   NVIDIA plugin time-slicing 实验      模型注册 ModelRegistry + 一键部署
第 5 周   DCGM Exporter + Prometheus 采集      MVP：模型部署 + 多租户 CRUD
第 6 周   Grafana 大盘（TTFT/TPOT/KV 命中率）  GPU 资源池模型设计（显存/算力/带宽）
第 7 周   （机动/补课）                        PlacementStrategy bin-packing 调度器
第 8 周   Volcano 批调度器（了解）             vGPU 切片分配 VgpuAllocator
第 9 周   昇腾 K8s 体系：MindX DL（文档级）    v1.0：资源池化 + 切片 + 监控大盘
第 10 周  海光/摩尔线程生态（文档级）           GpuProvider 抽象 + NvidiaProvider
第 11 周  异构差异对比笔记                     Ascend/Hygon/Moore mock 实现跑通
第 12 周  （机动）                             多租户配额 QuotaManager
第 13 周  Serverless AI：KNative/KEDA（了解）  ScaleController 缩容至 0
第 14 周  冷启动预热原理（warm pool）          WarmupManager 预热实现
第 15 周  ─                                    压测 + GPU 利用率数据采集
第 16 周  ─                                    v2.0：弹性伸缩 + 异构适配收尾
第 17 周  ─                                    README + 架构图 + 设计文档
第 18 周  必考题串联复盘                        交付冲刺
```

> Java 线进度慢就整体延后，K8s 理论线可穿插压缩。**异构适配（第 9-11 周）是 2026 招聘主命题，设计思路 > 跑通细节**——面试官要的是你能不能讲清"统一抽象怎么屏蔽差异"。

---

## 二、Part A：K8s + GPU 调度基础（第 1-8 周，理论 + 本地动手）

### 第 1 周：K8s 核心对象快速过（不从头学）

你有 6 年 Java 分布式底子，K8s 概念对你不难，按"类比 Java 微服务"的方式快速过：

| K8s 概念 | Java 微服务类比 | 一句话理解 |
|---|---|---|
| Pod | 一个服务实例 | 最小调度单元，可含多容器 |
| Deployment | 无状态服务 + 滚动发布 | 管理 Pod 副本数 |
| Node | 一台宿主机 | 调度的"格子" |
| Service/Ingress | Nginx/网关 | 服务发现与暴露 |
| ConfigMap/Secret | Apollo/Nacos 配置 | 配置与密钥注入 |
| PV/PVC | 共享存储挂载 | 模型文件挂载的关键（vLLM 权重挂载靠它） |

**资源**：官方中文文档过一遍概念篇（https://kubernetes.io/zh-cn/docs/concepts/）。
**自测**：能说清"部署一个 vLLM 服务在 K8s 里，需要哪些对象，各干什么"。

### 第 2 周：GPU 如何上 K8s —— Device Plugin 机制（面试必问）

这是本阶段的底层原理，务必吃透"为什么需要它"：

1. **问题**：K8s 原生只认 CPU/内存，GPU 是"extended resource"（扩展资源），kubelet 自己发现不了。
2. **解法**：NVIDIA 在每个 GPU 节点跑一个 **Device Plugin**（DaemonSet），负责：① 上报本节点 GPU 数量到 API Server；② 监听 Pod 的 GPU 请求，做设备分配与健康检查。
3. **请求方式**：Pod 里声明 `resources: limits: nvidia.com/gpu: 1`。

**进阶概念（按优先级）**：
- **Time-Slicing（时间片）**：一张卡被多个 Pod "共享"（并发跑但算力分时）——**消费卡也能玩，是你本地/租卡实验的关键**。
- **MIG（Multi-Instance GPU）**：硬件级显存+算力隔离切片，**只有 A100/A30/H100 等专业卡支持**（4090 不支持，文档级了解即可）。
- **vGPU（商业方案）**：NVIDIA 授权 + 驱动级虚拟化，个人搞不到，知道定位即可。

**自测**：能答"Device Plugin 解决什么问题""Time-Slicing 和 MIG 的区别（软件分时 vs 硬件隔离）"。

### 第 3 周：本地 K8s 环境 + 假 GPU（省钱核心，全程免费）

个人没有 K8s+GPU 集群很正常，**业界开发 AI 平台也大量用 fake GPU 方案**：

1. Mac 上装 Docker（推荐 OrbStack）+ **kind**（K8s in Docker）：

```bash
brew install kind
kind create cluster --name ai-lab
kubectl get nodes
```

2. 部署 **fake-gpu-operator**（run-ai 开源，专门给没有 GPU 的集群伪造 GPU 资源和指标）：

```bash
helm repo add fake-gpu-operator https://run-ai.github.io/fake-gpu-operator/
helm install fake-gpu-operator fake-gpu-operator/fake-gpu-operator
# 之后节点会出现 nvidia.com/gpu 资源，DCGM 指标也是模拟的
```

> 这一步的价值：你的 Java 平台后续所有"调度/分配/监控"逻辑，都能在这个假 GPU 集群上**真跑真测**，一分钱不花。面试讲出来还是加分项（"我用 fake-gpu-operator 搭建了开发环境"体现工程素养）。

### 第 4 周：time-slicing 实验（租卡时做，或本地 fake 环境做）

在 fake 环境或真卡上，给 NVIDIA device plugin 开时间片共享：

```yaml
# time-slicing 配置片段（ConfigMap: nvidia-plugin）
version: v1
sharing:
  timeSlicing:
    resources:
    - name: nvidia.com/gpu
      replicas: 4   # 1 张卡当 4 张"逻辑卡"卖
```

**观察**：`kubectl describe node` 里 `nvidia.com/gpu` 从 1 变 4；起 4 个 Pod 都能分到"半张卡"。
**思考题**：时间片共享的风险是什么？（无显存隔离 → 一个 Pod OOM 会拖死同卡邻居 → 引出为什么生产要 MIG/驱动级隔离）——这就是面试"显存级隔离"的答题素材。

### 第 5 周：指标采集 —— DCGM Exporter + Prometheus

1. **采集端**：NVIDIA 官方 **DCGM Exporter**（DaemonSet，暴露 GPU 利用率/显存/温度为 Prometheus 格式）：

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm install dcgm gpu-helm-charts/dcgm-exporter
kubectl port-forward svc/dcgm-exporter 9400:9400
curl localhost:9400/metrics   # 看到 DCGM_FI_DEV_GPU_UTIL 等指标
```

2. **真机验证（可选，省钱方案）**：4070 Ti 机器若是 Linux/WSL2，装 Docker 跑一个 `dcgm-exporter` + Prometheus，就能**免费采集真实 GPU 指标**；不行就全靠 fake 指标。

3. **推理业务指标**：vLLM 自带 `/metrics` 端点（Prometheus 格式），重点指标：

| 指标 | 含义 |
|---|---|
| `vllm:time_to_first_token_seconds` | TTFT（P99 面试高频） |
| `vllm:time_per_output_token_seconds` | TPOT |
| `vllm:gpu_cache_usage_perc` | **KV Cache 显存利用率** |
| `vllm:num_requests_running/waiting` | 运行/排队请求数 |

**自测**：能画出"DCGM（硬件指标）+ vLLM /metrics（业务指标）→ Prometheus → Grafana"的采集链路。

### 第 6 周：Grafana 监控大盘（可交付物）

```bash
helm install prometheus prometheus-community/kube-prometheus-stack
# Grafana 导入 NVIDIA 官方 DCGM 大盘模板（ID: 12239），再自建 vLLM 业务大盘
```

**自建大盘面板（对照路线图要求）**：GPU 利用率、显存占用、KV Cache 命中率/利用率、TTFT P99、TPOT P99、排队深度。
**交付**：截图存档——这是《全链路监控大盘》交付物的雏形，也是"API 网关 → 推理引擎 → KV Cache"拓扑的落地。

### 第 8 周：Volcano 批调度器（了解级）

- K8s 默认调度器是"单 Pod 视角"，**vLLM 多机推理（TP=8）要求 8 个 Pod 同起同落**，默认调度器可能部分调度成功 → 死锁。
- **Volcano** 提供 gang scheduling（成组调度），是 AI 训练/推理的社区标准答案。
- 精度要求：知道"gang scheduling 解决什么问题"即可，不用深入源码。

---

## 三、Part B：Java 平台开发 `ai-orchestrator`（第 3-18 周主线）

> 完整技术方案见路线图 4.5 节（目录结构 + 模块划分）。这里给执行顺序与每步验收。

### MVP（第 3-5 周）：模型一键部署 + 多租户

1. **项目骨架**：Java 17 + Spring Boot 3，Maven 多模块（`orchestrator-core` / `orchestrator-api`）。
2. **K8s 客户端选型**：用 **fabric8 kubernetes-client**（https://github.com/fabric8io/kubernetes-client，比官方 client 顺手）：

```java
// 核心：把"部署一个 vLLM 模型"翻译成 K8s Deployment + Service
KubernetesClient client = new KubernetesClientBuilder().build();
// 1. 读模型注册表（model name -> 镜像 + 默认参数 + 显存需求）
// 2. createDeployment（vLLM 容器，挂模型 PVC，requests: nvidia.com/gpu=N）
// 3. createService（ClusterIP，暴露 8000）
```

3. **ModelRegistry**：模型元数据表（名称/版本/量化精度/显存需求/默认并发）。
4. **多租户**：tenant 表 + 每租户 namespace 隔离（K8s namespace 天然适配多租户）。

**MVP 验收**：调用平台 REST API `POST /api/v1/models/deploy`，fake GPU 集群里真的出现 vLLM Deployment + Service；删部署、查状态、多租户互见不到对方的模型。

### v1.0（第 6-9 周）：GPU 资源池化 + 切片 + 大盘

1. **统一资源模型**（`scheduler/GpuResourcePool`）：每张逻辑卡 = { 显存 GB, 算力 TFLOPS, 带宽, 所属节点, 已分配租户 }。
2. **PlacementStrategy（bin-packing）**：
   - 策略：优先填满已有节点（减少碎片）→ 同节点内优先选剩余显存最大的卡。
   - 实现一个 `PlacementStrategy` 接口 + `BestFitStrategy` 默认实现，**写单元测试**（造 20 个请求场景断言放置结果）。
3. **VgpuAllocator**：显存级切片（如 24G 卡切 4×6G），分配记录持久化（MySQL/Postgres）。
4. **监控接入**：平台读 Prometheus API（简单 HTTP 查询），聚合成租户维度的用量报表。

**v1.0 验收**：fake GPU 集群 + 时间片下，平台能把 12 个不同显存需求的部署请求合理铺到 4 张"逻辑卡"上；Grafana 能看到每租户的 GPU 用量。**目标数据：资源填充率（碎片率）从 naive 策略的 ~60% 提到 85%+——这就是"GPU 利用率 30% → 75%"故事的第一半。**

### v2.0（第 10-16 周）：异构适配 + 弹性伸缩（简历最大亮点）

#### 异构算力抽象层（第 10-11 周，主命题）

1. **统一接口**（先设计后实现，面试重点考设计）：

```java
public interface GpuProvider {
    String vendor();                                  // NVIDIA / ASCEND / HYGON / MTHREADS
    List<GpuDeviceInfo> discover(Node node);          // 设备发现
    ResourceSpec getCapability(GpuDeviceInfo dev);    // 能力（显存/算力/支持的量化）
    void attachToPod(PodSpec spec, GpuDeviceInfo dev);// 挂载设备资源
    MetricSnapshot getMetrics(GpuDeviceInfo dev);     // 指标采集
}
```

2. **四个实现的差异点（做成对比笔记，面试直接用）**：

| 厂商 | 芯片/生态 | K8s 组件 | 关键差异 |
|---|---|---|---|
| NVIDIA | CUDA | k8s-device-plugin + DCGM | 生态最全，MIG/time-slicing |
| 华为昇腾 | CANN + MindIE | **Ascend Device Plugin + MindX DL + Volcano** | 需要 NPU 切片（vNPU），镜像内含 CANN 工具链 |
| 海光 | DCU（ROCm 兼容） | 类 NVIDIA plugin（HIP 生态） | 软件栈兼容 AMD ROCm |
| 摩尔线程 | MUSA | 自研 plugin | CUDA 语法兼容层，生态最弱但成本最低 |

3. **昇腾重点投入**（国产刚需，面试官最常追问）：
   - 精读昇腾社区 K8s 方案 **mind-cluster**（https://gitee.com/ascend/mind-cluster）：Ascend Device Plugin + Volcano 调度器增强。
   - 理解 MindIE（昇腾推理引擎）与 vLLM 的关系：MindIE-Service 提供类 OpenAI 接口，平台侧**接口统一、后端可替换**。
   - 实操降级：没有 NPU 就用 mock（单元测试覆盖 discover/allocate 逻辑），能讲清组件拓扑即可。

4. **抽象层价值闭环**：平台上部署模型时只声明 `{model: "qwen2.5-7b", backend: "auto"}`，调度器按"租户标签 + 成本权重 + 库存"选择 provider → **这是"统一抽象屏蔽硬件差异"的完整故事**。

#### 弹性伸缩（第 12-14 周）

1. **QuotaManager（第 12 周）**：租户额度 = { 显存上限, 并发上限, 优先级 }；超额→排队或降级（联动阶段二 Rust 网关的限流）。
2. **ScaleController（第 13 周）**：
   - 指标驱动：Prometheus 查询 `vllm:num_requests_waiting` P95 > 阈值 → 扩容；< 低水位持续 N 分钟 → **缩容至 0**（Deployment replicas=0，显存释放回池）。
   - 对比了解 KNative/KEDA 怎么做 scale-to-0（面试能对比"自研 vs 开源"）。
3. **WarmupManager（第 14 周）——3 秒冷启动的真相**：
   - 模型加载要几十秒，纯 scale-to-0 不可能 3 秒。
   - **正确姿势：warm pool（预热池）**——保底常驻 N 个已加载实例（小模型），请求洪峰先打 warm pool，同时异步扩容新实例；"3 秒"指**从预热实例接流量到新实例就绪的切换时间**。
   - 预热动作：启动后立刻发 1 条 dummy 请求，触发 CUDA graph 捕获 + 权重加载。

**v2.0 验收**：压测流量爬坡时平台自动扩容；流量回落后 5 分钟内缩容至 0；再次打流量时 warm pool 接住，**P99 首字延迟无毛刺**（录屏/截图存证）。

### 第 15-18 周：数据 + 交付

1. **压测出数（面试数据的来源）**：
   - 用 fake 环境测"调度填充率"；真机数据（可选）：4070 Ti 若可跑 Linux/WSL2 + Docker + dcgm-exporter，采一组真实 GPU 利用率数据。
   - 记录：naive 放置 vs bin-packing 的碎片率、缩容前后显存占用、warm pool 命中率。
2. **《异构算力调度层设计文档》**（路线图要求交付物）：统一资源模型 → 四厂商差异 → 调度策略 → 弹性伸缩。配一张架构图（用 drawio/mermaid 画）。
3. **README**：一句话定位 + 架构图 + GIF/截图 + 关键数据表。

---

## 四、阶段三总验收（对照路线图 4.4）

- [ ] 平台能完成模型一键部署 + 多租户隔离 + 弹性伸缩（fake GPU 集群真实跑通）
- [ ] 能画出"API 网关 → 推理引擎 → KV Cache"全链路监控拓扑，且大盘有截图
- [ ] 能讲清 NVIDIA / 昇腾 / 海光 / 摩尔线程的调度差异与统一抽象思路（`GpuProvider` 设计）
- [ ] 能答"如何把 GPU 利用率从 30% 提升到 75%"（池化 + bin-packing + time-slicing/MIG + 弹性伸缩 + warm pool，逐条对应你平台里的模块）
- [ ] 能答"Device Plugin 是什么、为什么需要""gang scheduling 解决什么问题"
- [ ] 能讲清 warm pool 方案下"3 秒冷启动"到底指什么（体现工程诚实与深度）

---

## 五、参考文章链接

### K8s 与 GPU 调度
- K8s 官方中文文档：https://kubernetes.io/zh-cn/docs/home/
- NVIDIA k8s-device-plugin（含 time-slicing 配置文档）：https://github.com/NVIDIA/k8s-device-plugin
- NVIDIA MIG 用户指南：https://docs.nvidia.com/datacenter/tesla/mig-user-guide/
- DCGM Exporter：https://github.com/NVIDIA/dcgm-exporter
- fake-gpu-operator（无卡开发 K8s GPU 平台的利器）：https://github.com/run-ai/fake-gpu-operator
- Volcano 批调度系统：https://volcano.sh/zh/docs/
- kind（本地 K8s）：https://kind.sigs.k8s.io/

### 异构算力
- 昇腾社区（CANN/MindIE 文档）：https://www.hiascend.com/document
- 昇腾 K8s 方案 mind-cluster（Ascend Device Plugin）：https://gitee.com/ascend/mind-cluster
- 海光 DCU 与 ROCm 生态：https://rocm.docs.amd.com/
- 摩尔线程 MUSA：https://www.mthreads.com/

### 监控与弹性
- vLLM 官方 metrics 说明：https://docs.vllm.ai/en/latest/serving/metrics.html
- Prometheus 官方文档：https://prometheus.io/docs/introduction/overview/
- Grafana 文档：https://grafana.com/docs/
- KEDA（基于事件驱动伸缩）：https://keda.sh/
- KNative Serving（scale-to-0 参考）：https://knative.dev/docs/

### Java 侧
- Spring Boot 3：https://spring.io/projects/spring-boot
- fabric8 kubernetes-client：https://github.com/fabric8io/kubernetes-client
- Prometheus Java client：https://github.com/prometheus/client_java

---

## 六、省钱与避坑

1. **本阶段几乎不花钱**：Mac（Java + kind + fake-gpu-operator）搞定 95% 开发。可选支出：① 4070 Ti 机器（Linux/WSL2）跑真实 dcgm-exporter 采真数据，0 元；② 昇腾真机短租（华为云 910B 约 10-20 元/时）——**预算紧就跳过，mock + 文档级足够支撑面试**，别为"真机情结"花几百块。
2. **最大风险：陷入"造 K8s"**。你是做"平台"，不是做"K8s 替代品"——调度/配额/伸缩逻辑自研，设备发现/挂载/PV 一律复用 K8s 原生与开源组件。
3. **第二风险：异构适配做太深**。面试要的是"统一抽象设计 + 厂商差异对比"，NvidiaProvider 跑通 + 三个 mock 实现即可，别在 CANN 环境安装上耗一周。
4. 每周周记继续（阶段二的方法论），第 18 周汇总成复盘文档。
5. 平台的 REST API 设计要对齐你的 Java 审美（DTO/统一异常/幂等），**这个项目同时是你"资深 Java 架构能力"的证明**，代码质量直接决定面试印象。
