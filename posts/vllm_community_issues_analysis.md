# vLLM 社区近期 100 个未关闭重要 Issues 分析

> 数据来源：[github.com/vllm-project/vllm/issues](https://github.com/vllm-project/vllm/issues)
> 抓取时间：2026 年 10 月 13 日
> 筛选条件：`is:issue is:open`，按创建时间倒序取前 100 个
> 当前未关闭 issue 总数：**2703**

---

## 一、总体概览

近期 vLLM 社区的 open issues 集中反映了几个核心方向的痛点：

| 类别 | Issue 数量 | 严重程度 |
|------|:---------:|:--------:|
| 推测解码 (Speculative Decoding) | 12 | 高 |
| DeepSeek-V4.1 模型支持 | 10 | 高 |
| API 兼容性（Responses API / Anthropic API / Tool Calling） | 14 | 中高 |
| 量化 / 精度问题 | 9 | 高 |
| KV Cache / 连接器 | 11 | 高 |
| CUDA Graph / 编译 | 6 | 高 |
| 多模态 | 6 | 中高 |
| ROCm / AMD GPU | 6 | 中高 |
| 性能优化 | 8 | 中 |
| LoRA | 3 | 中 |
| Intel XPU | 2 | 中 |
| 安装 / 兼容性 | 7 | 中 |
| 新硬件支持 (SM120 / GB10 / Jetson) | 5 | 中高 |
| 其他 (RFC / 文档 / CPU 后端等) | 8 | 低-中 |

---

## 二、分类详细分析

### 2.1 推测解码 (Speculative Decoding) — 12 个

涉及 DSpark、Eagle、MTP、Dynamic SD 等多种推测解码方案。

#### 核心问题

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63439](https://github.com/vllm-project/vllm/issues/63439) | [RFC] Reduce CPU overhead in Model Runner V2 for long-context P/D serving | P/D 分离部署下 decode GPU 空闲 17%，CPU 侧 per-request Python 列表/GC 成为瓶颈 | 1 |
| [#63359](https://github.com/vllm-project/vllm/issues/63359) | [RFC] Use session Agent Hints to reuse prompt tokens and block hashes | Agent 场景每次对话都要重新 tokenize + 构建 block hash，即使 KV 已缓存 | 0 |
| [#63157](https://github.com/vllm-project/vllm/issues/63157) | [Spec Decode][DeepSeek-V4.1] Support DSpark with pipeline parallelism | DSpark 在 PP 下被 capability check 拒绝，引擎无法初始化 | 1 |
| [#63128](https://github.com/vllm-project/vllm/issues/63128) | [Bug][Spec Decode] MTP draft 在 PP 下 embedding 未初始化导致接受率崩溃 | 0 | 1 |
| [#63152](https://github.com/vllm-project/vllm/issues/63152) | [Tracking] Dory looped model: per-request loops, spec decode, GB10 perf | Dory 模型推测解码、循环、GB10 性能跟踪 | 0 |
| [#63004](https://github.com/vllm-project/vllm/issues/63004) | [RFC] add eagle4 spec decode | 请求集成 EAGLE-4 推测解码 | 1 |
| [#62939](https://github.com/vllm-project/vllm/issues/62939) | [Bug][MRV2] DSpark + adaptive verification 启动崩溃 | max_num_batched_tokens < max_num_seqs × draft rows 时 device assert | 3 |
| [#62793](https://github.com/vllm-project/vllm/issues/62793) | [Bug] MRV2 adaptive verification 降低 Eagle3 接受率 | draft-prefill CUDA graph padding 不匹配 | 0 |
| [#62818](https://github.com/vllm-project/vllm/issues/62818) | [RFC] Preserve output distribution under structured-output + spec decode | 结构化输出 + 推测解码时输出分布被破坏 | 3 |
| [#62694](https://github.com/vllm-project/vllm/issues/62694) | [Feature] Support model-based drafters (target hidden states) via custom_class | custom_class 不支持需要 target hidden states 的 drafter (DFlash, Orthrus) | 0 |
| [#62740](https://github.com/vllm-project/vllm/issues/62740) | [Feature] Measure the Dynamic SD table instead of writing it by hand | Dynamic SD 表需手写，用户无法知道是否适配自己的 GPU/模型 | 0 |
| [#62707](https://github.com/vllm-project/vllm/issues/62707) | [Bug] v0.25.1 流式输入超 max_model_len 产生负 token count 崩溃 V2 runner | 1 |

#### 分析与解决方向

- **CPU 开销瓶颈 ([#63439](https://github.com/vllm-project/vllm/issues/63439), [#63363](https://github.com/vllm-project/vllm/issues/63363), [#63359](https://github.com/vllm-project/vllm/issues/63359))**：长上下文 P/D 场景下，Python `list[int]` 持有 prompt token 的双份拷贝导致 GC 暂停随 prompt 长度线性增长。解决方向：(1) 用 `array.array` 或 `torch.Tensor` 替代 Python list 存储 token ids；(2) 引入 session-level agent hints 复用已缓存的 block hash，避免重复 tokenize。
- **PP 兼容性 ([#63157](https://github.com/vllm-project/vllm/issues/63157), [#63128](https://github.com/vllm-project/vllm/issues/63128))**：DSpark 和 MTP 在 pipeline parallelism 下存在根本性限制——`_maybe_share_embeddings` 只在 `pp_world_size==1` 时共享，draft-only checkpoint 的 embedding 未初始化。需要扩展 embedding 共享逻辑到 PP 场景。
- **Adaptive verification ([#62939](https://github.com/vllm-project/vllm/issues/62939), [#62793](https://github.com/vllm-project/vllm/issues/62793))**：MRV2 的 adaptive verification 在 `max_num_batched_tokens` 较小时触发 device-side assert；同时 draft-prefill CUDA graph 与 verifier 的 padding 不一致导致接受率下降。需要在 graph capture 阶段统一 padding 策略。
- **EAGLE-4 集成 ([#63004](https://github.com/vllm-project/vllm/issues/63004))**：社区请求将 EAGLE-4 加入 vLLM，需评估其 API 与现有 `custom_class` 机制的兼容性。
- **结构化输出 + 推测解码 ([#62818](https://github.com/vllm-project/vllm/issues/62818))**：当前 `StructuredOutputManager` 的三步流程在同步/异步调度下可能破坏输出分布，需要设计在 grammar validation 不 commit state 的前提下保持分布正确性。

---

### 2.2 DeepSeek-V4.1 模型支持 — 10 个

DeepSeek-V4.1 是近期重点支持的新模型，围绕它有大量 bug 和兼容性问题。

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63343](https://github.com/vllm-project/vllm/issues/63343) | [Bug][SM120] enable_adaptive_verification 在 SM120 被拒绝 | device-query-lens 只支持 SM90/SM100 | 4 |
| [#63340](https://github.com/vllm-project/vllm/issues/63340) | [Feature][ROCm] Mono decode: enable MI350P (gfx950, 128 CUs) | 128 CU 的 MI350P 需要不同 grid 配置 | 2 |
| [#63276](https://github.com/vllm-project/vllm/issues/63276) | [Bug] DS V4.1 Flash1017 输出错误数字 | 数字序列 echo 不稳定 | 1 |
| [#63250](https://github.com/vllm-project/vllm/issues/63250) | [Bug] DeepSeek-V4.1-Flash(-1017) 返回 U+FFFD | 12 token prompt 返回替换字符，TP8+EP, 8xH20 | 1 |
| [#63240](https://github.com/vllm-project/vllm/issues/63240) | [Question] Qwen4Exp PLE n-gram table 密集 bf16 材料化 | 能否保持在 pinned host memory 中压缩 | 1 |
| [#62965](https://github.com/vllm-project/vllm/issues/62965) | [Feature] Capture DSv4.1 decoder replay graphs on model graphs' stream | replay graph 在独立 stream 捕获，不共享 graph pool | 0 |
| [#62935](https://github.com/vllm-project/vllm/issues/62935) | [Bug] DS-V4.1-Flash-1017 GB300 长 agent 回复陷入重复循环 | 1 |
| [#62811](https://github.com/vllm-project/vllm/issues/62811) | [Bug] DS-V4.1 chunked prefill 最后一块走 PIECEWISE CUDA graph 输出错误 | 0 |
| [#62839](https://github.com/vllm-project/vllm/issues/62839) | [RFC] Fuse DSpark confidence-head GEMV into single TileLang kernel | 融合后 kernel 延迟降低 ~1.9x | 1 |
| [#62939](https://github.com/vllm-project/vllm/issues/62939) | (见推测解码) DSpark + adaptive verification 启动崩溃 | 3 |

#### 分析与解决方向

- **输出正确性 ([#63276](https://github.com/vllm-project/vllm/issues/63276), [#63250](https://github.com/vllm-project/vllm/issues/63250), [#62935](https://github.com/vllm-project/vllm/issues/62935), [#62811](https://github.com/vllm-project/vllm/issues/62811))**：多个 issue 报告 V4.1-Flash 在不同配置下输出错误——数字漂移、U+FFFD 替换字符、重复循环、chunked prefill 最后一块 CUDA graph 路径错误。这些指向 **piecewise CUDA graph 捕获/回放逻辑** 的根本性缺陷，尤其在边界条件（最后一块 prefill chunk 很小）下。解决方向：(1) 对 PIECEWISE graph 的最后一块强制使用 eager 或 FULL 模式；(2) 修复 split-KV combine kernel 的 tile scheduler 使用 capture-time dummy batch 的问题 ([#62626](https://github.com/vllm-project/vllm/issues/62626))。
- **新硬件适配 ([#63343](https://github.com/vllm-project/vllm/issues/63343), [#63340](https://github.com/vllm-project/vllm/issues/63340))**：SM120 (Blackwell RTX PRO 6000) 的 adaptive verification 被 device-query-lens 门控拒绝；MI350P (gfx950, 128 CUs) 的 mono decode kernel 需要调整 grid width。解决方向：扩展 `resolve_tiling()` 支持 128-CU 架构，移除 SM 级别的硬门控改为 capability-based 检查。
- **性能优化 ([#62965](https://github.com/vllm-project/vllm/issues/62965), [#62839](https://github.com/vllm-project/vllm/issues/62839), [#63240](https://github.com/vllm-project/vllm/issues/63240))**：decoder replay graph 应共享 model graphs 的 stream 以复用 graph pool；DSpark confidence-head GEMV 可融合为单个 TileLang kernel 延迟降低近 2x；Qwen4Exp PLE n-gram table 可保持在压缩 pinned host memory 中减少 GPU 显存占用。

---

### 2.3 API 兼容性（Responses API / Anthropic API / Tool Calling）— 14 个

vLLM 的 OpenAI 兼容层和 Anthropic 兼容层存在大量协议偏差和 parser 一致性问题。

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63358](https://github.com/vllm-project/vllm/issues/63358) | Hermes tool parser: streaming vs non-streaming 不一致 | 有效 tool call 后跟 malformed `<tool` 区域时两条路径结果不同 | 0 |
| [#63356](https://github.com/vllm-project/vllm/issues/63356) | Chat derender 忽略 `--response-role`, `add_generation_prompt: false`, `echo` | 总是返回 `role: "assistant"` | 0 |
| [#63346](https://github.com/vllm-project/vllm/issues/63346) | Streaming chat 对 `tool_choice: "none"` 仍返回 tool calls | batch 正确丢弃，stream 不丢弃 | 1 |
| [#63309](https://github.com/vllm-project/vllm/issues/63309) | Hermes tool parser streaming 丢弃 `<<` 后所有内容 | prose 中出现 `<<` 标记时静默丢弃 | 1 |
| [#63331](https://github.com/vllm-project/vllm/issues/63331) | Anthropic /v1/messages: inline system 消息合并导致 prefix cache 失效 | Claude Code 每轮都 miss prefix cache | 1 |
| [#63310](https://github.com/vllm-project/vllm/issues/63310) | Responses API 拒绝无 `type` 字段的 assistant input items | 客户端 (ECA/Emacs) 发送的历史格式被 400 拒绝 | 0 |
| [#63185](https://github.com/vllm-project/vllm/issues/63185) | Responses API 拒绝 `include: ["web_search_call.*"]` | 虽然 `web_search` tool 被接受 | 0 |
| [#63184](https://github.com/vllm-project/vllm/issues/63184) | Responses API: `reasoning.summary` 总是返回空数组 | 接受并回显但 reasoning items 的 summary 始终为 `[]` | 1 |
| [#63183](https://github.com/vllm-project/vllm/issues/63183) | Responses API: completed responses 不带 `completed_at` 字段 | 0 |
| [#63130](https://github.com/vllm-project/vllm/issues/63130) | Strict serde 拒绝 `reasoning` + `reasoning_content` 重复字段 | Agent harness 回显两个字段时 Rust 前端 400 | 3 |
| [#62830](https://github.com/vllm-project/vllm/issues/62830) | [Bug] Anthropic API /v1/messages 忽略 assistant prefill | 关闭最后一轮 assistant 并新开一轮 | 1 |
| [#62639](https://github.com/vllm-project/vllm/issues/62639) | [Bug] Responses API: `tool_choice=allowed_tools` 不工作 | 3 |
| [#62641](https://github.com/vllm-project/vllm/issues/62641) | [Usage] 变更工具集时避免完整 prompt 重处理 | 0 |
| [#62743](https://github.com/vllm-project/vllm/issues/62743) | K2 Horizon reasoning parser 只接受请求 effort 的结束标签 | 模型用 `</ifm\|think>` 关闭 medium/low thinking 时 reasoning 泄漏 | 0 |

#### 分析与解决方向

- **Streaming vs Non-streaming 一致性 ([#63358](https://github.com/vllm-project/vllm/issues/63358), [#63346](https://github.com/vllm-project/vllm/issues/63346), [#63309](https://github.com/vllm-project/vllm/issues/63309))**：多条 issue 都指向同一根因——batch 路径通过 `assemble_chat_message_fields()` 过滤 tool calls，而 streaming 路径绕过了这个逻辑。解决方向：抽取共享的 tool-call 过滤逻辑，确保两条路径使用同一套规则。
- **Responses API 字段缺失 ([#63183](https://github.com/vllm-project/vllm/issues/63183), [#63184](https://github.com/vllm-project/vllm/issues/63184), [#63185](https://github.com/vllm-project/vllm/issues/63185), [#63310](https://github.com/vllm-project/vllm/issues/63310))**：`completed_at`、`reasoning.summary`、`web_search_call.*` include 等多个 OpenAI Responses API 字段未实现或实现不完整。需要逐项对齐 OpenAI API 规范，补全 protocol model 字段。
- **Anthropic API 兼容性 ([#63331](https://github.com/vllm-project/vllm/issues/63331), [#62830](https://github.com/vllm-project/vllm/issues/62830))**：(1) inline system 消息合并到 leading system prompt 导致 prefix cache 失效——需要在合并时保留可缓存边界；(2) assistant prefill 被忽略——`/v1/messages` 应支持 prefill 语义，保留最后一轮 assistant turn 不闭合。
- **Rust 前端 serde 严格性 ([#63130](https://github.com/vllm-project/vllm/issues/63130))**：`reasoning_content` 作为 `reasoning` 的 serde alias 导致同时出现两个字段时被 serde 拒绝。解决方向：改为 `#[serde(alias = "reasoning_content", alias = "reasoning")]` 或在反序列化前做 dedup。

---

### 2.4 量化 / 精度问题 — 9 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63043](https://github.com/vllm-project/vllm/issues/63043) | Native FlashInfer BF16 Q + 非 unit q_scale 产生 garbage output | FP8 KV 下 q_scale 被错误应用 | 0 |
| [#62820](https://github.com/vllm-project/vllm/issues/62820) | FlashInfer fp4_gemm autotune 改变 temperature=0 token | Qwen3.8-27B NVFP4 (SM120) | 1 |
| [#62851](https://github.com/vllm-project/vllm/issues/62851) | Qwen3.6 GDN greedy DFlash2 批次依赖数值偏差 + preemption/recompute | 3 |
| [#62803](https://github.com/vllm-project/vllm/issues/62803) | D3 checkpoint loaded as Qwen3.5 产生均匀 decision scores | 1 |
| [#62762](https://github.com/vllm-project/vllm/issues/62762) | [Performance] XPU block FP8: W8A16 比 W8A8 快 64% | decode 35→58 tok/s | 1 |
| [#63240](https://github.com/vllm-project/vllm/issues/63240) | (见 DSv4.1) Qwen4Exp PLE n-gram table 密集 bf16 | 1 |
| [#63000](https://github.com/vllm-project/vllm/issues/63000) | 长上下文生成中途以 `<\|im_end\|>` 停止 | Qwen3.8-27B fp8 KV cache | 1 |
| [#62707](https://github.com/vllm-project/vllm/issues/62707) | (见推测解码) v0.25.1 流式输入超限崩溃 | 1 |
| [#62677](https://github.com/vllm-project/vllm/issues/62677) | v0.32.0 官方镜像: deepep_v2 ZeroDivisionError | ibstat 带宽检测失败 | 0 |

#### 分析与解决方向

- **FlashInfer q_scale bug ([#63043](https://github.com/vllm-project/vllm/issues/63043))**：FP8 KV cache 下，当 Q 为 BF16/FP16 且未经 FP8 量化时，`_q_scale_float` 仍被传入 attention kernel，非 unit scale 错误改变了 attention 输出。修复方向：在 Q 不是 FP8 时强制 `q_scale=1` 或跳过 scale 传递。
- **NVFP4 autotune 非确定性 ([#62820](https://github.com/vllm-project/vllm/issues/62820))**：FlashInfer fp4_gemm 的 autotune 会选择不同 kernel 配置，导致 temperature=0（greedy）下 token 发生变化。需要固定 autotune cache 或在 greedy 模式下跳过 autotune。
- **GDN fp16 溢出 ([#62851](https://github.com/vllm-project/vllm/issues/62851), [#62781](https://github.com/vllm-project/vllm/issues/62781))**：Qwen3.5/3.6 GDN 在 fp16 下长上下文 needle 测试 0/9，溢出导致 attention 输出错误。已有修复 ([#54146](https://github.com/vllm-project/vllm/issues/54146)/[#62119](https://github.com/vllm-project/vllm/issues/62119)) 但未覆盖 XPU 路径。需要在 GDN linear attn 路径加入 fp32 中间计算或 clamp。
- **W8A16 vs W8A8 ([#62762](https://github.com/vllm-project/vllm/issues/62762))**：Intel XPU 上 block FP8 的 W8A8 路径比 W8A16 慢 40%。解决方向：在 XPU 上默认使用 W8A16 路径（`apply_input_quant=False`）。

---

### 2.5 KV Cache / 连接器 — 11 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63336](https://github.com/vllm-project/vllm/issues/63336) | [RFC] MoRIIO transfer contract: safe KV reclamation for P/D | READ 模式 60s 后释放 block 但 decode 可能仍在读 | 1 |
| [#63149](https://github.com/vllm-project/vllm/issues/63149) | [Tracking] KV event correctness and compatibility | 跨架构/推测解码 KV 事件可靠性跟踪 | 8 |
| [#63037](https://github.com/vllm-project/vllm/issues/63037) | [Bug] Reused OffloadKey 从延迟 BlockStored 事件丢失 metadata | 0 |
| [#63006](https://github.com/vllm-project/vllm/issues/63006) | [RFC] Reduce descriptor/WR overhead for Mooncake KV transfer | Mooncake 比 NIXL 慢 35-67% | 4 |
| [#63058](https://github.com/vllm-project/vllm/issues/63058) | [Feature] keep_tags for CuMemAllocator.sleep | 保留 kv_cache 映射只 offload weights | 2 |
| [#62862](https://github.com/vllm-project/vllm/issues/62862) | [Performance] MultiConnector 禁用 divergent local hybrid cache hits | NixlConnector + OffloadingConnector 组合时 decode 侧本地缓存命中被禁用 | 0 |
| [#62931](https://github.com/vllm-project/vllm/issues/62931) | [Bug] Tiered KV offload: 二级 tier hit 大于 CPU tier 时加载 0 | 0 |
| [#62808](https://github.com/vllm-project/vllm/issues/62808) | [Bug] LMCacheConnectorV1 不转发 shutdown()，LMCache 不停止 | EngineCore 被 force-kill | 2 |
| [#62810](https://github.com/vllm-project/vllm/issues/62810) | [RFC] Expose allocator-issued KV block generations to async connectors | block_id 复用后无法验证 DMA 目标有效性 | 0 |
| [#62866](https://github.com/vllm-project/vllm/issues/62866) | [Doc] MooncakeStoreConnector 文档未提及 kv_load_failure_policy | 默认 "fail" 导致 load 失败返回空响应 | 0 |
| [#63020](https://github.com/vllm-project/vllm/issues/63020) | (见安装) flint NVFP4 GGUF checkpoint serving 需 workarounds | 2 |

#### 分析与解决方向

- **KV 事件正确性 ([#63149](https://github.com/vllm-project/vllm/issues/63149), [#63037](https://github.com/vllm-project/vllm/issues/63037))**：这是一个 tracking issue，覆盖 attention block、recurrent checkpoint、partial hit、speculative rollback 等场景的 KV 事件可靠性。核心目标是让下游消费者（Dynamo、llm-d、vLLM Router）能准确判断哪些 cache entry 仍然驻留。需要系统性修复事件发布/恢复路径。
- **MoRIIO 安全释放 ([#63336](https://github.com/vllm-project/vllm/issues/63336))**：RDMA KV 传输中，READ 模式 prefill 在 60s 后释放 block 但 decode 可能仍在异步读取，导致读到其他请求的 KV。需要引入显式 ownership transfer contract——在 decode 确认完成前不释放。
- **Mooncake 带宽优化 ([#63006](https://github.com/vllm-project/vllm/issues/63006))**：大量小 transfer descriptor + 串行 WR 构建/post 导致 Mooncake 比 NIXL 慢 35-67%。优化方向：(1) 批量合并小 transfer 为大 WR；(2) 并行化 WR 构建与提交；(3) 减少 descriptor 数量。
- **Connector 生命周期 ([#62808](https://github.com/vllm-project/vllm/issues/62808), [#63037](https://github.com/vllm-project/vllm/issues/63037))**：LMCacheConnectorV1 未实现 `shutdown()`；OffloadKey 从延迟事件丢失 metadata。需要补全 connector 的生命周期管理方法，并确保事件顺序的正确性。
- **Block generation 暴露 ([#62810](https://github.com/vllm-project/vllm/issues/62810))**：block_id 复用后无法验证异步 DMA 目标有效性。建议为 `KVCacheBlock` 增加 generation number，让 connector 能验证 block 是否仍属于原始 allocation。

---

### 2.6 CUDA Graph / 编译 — 6 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#62811](https://github.com/vllm-project/vllm/issues/62811) | (见 DSv4.1) chunked prefill 最后一块走 PIECEWISE graph 输出错误 | 0 |
| [#63052](https://github.com/vllm-project/vllm/issues/63052) | [Bug] PIECEWISE capture 在废弃 stream 上残留 warmup activations | 0 |
| [#62626](https://github.com/vllm-project/vllm/issues/62626) | [Bug] `cudagraph_mode=FULL` + FA3 split-KV 返回错误 logits | combine kernel tile scheduler 从 capture-time dummy batch 选择 | 1 |
| [#62939](https://github.com/vllm-project/vllm/issues/62939) | (见推测解码) DSpark adaptive verification 启动崩溃 | 3 |
| [#62793](https://github.com/vllm-project/vllm/issues/62793) | (见推测解码) MRV2 adaptive verification 降低 Eagle3 接受率 | 0 |
| [#62865](https://github.com/vllm-project/vllm/issues/62865) | [Bug] Compile mode 2 + Transformers backend 产生错误 kernel | 0 |

#### 分析与解决方向

- **FULL graph + FA3 split-KV ([#62626](https://github.com/vllm-project/vllm/issues/62626))**：capture-time dummy batch 的 tile scheduler 与实际 runtime 的 split 数不匹配，导致 combine kernel 输出错误 logits。临时方案：设 `flash_attn_max_num_splits_for_cuda_graph=1` 关闭 captured graph 内的 split-KV。根本修复：在 graph replay 时重新选择 tile scheduler。
- **PIECEWISE 边界 ([#62811](https://github.com/vllm-project/vllm/issues/62811), [#63052](https://github.com/vllm-project/vllm/issues/63052))**：chunked prefill 的最后一块（可能只有 10 个 token）在 PIECEWISE graph 路径下输出错误；capture stream 被废弃后 warmup activations 未清理。需要在 capture 后正确释放临时资源，并对边界 chunk 使用不同路径。
- **Compile mode 2 ([#62865](https://github.com/vllm-project/vllm/issues/62865))**：Transformers backend 在 compile mode 2 下生成错误 kernel，可能是 inductor 的 kernel 融合与 vLLM 的自定义 op 产生冲突。

---

### 2.7 多模态 — 6 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63414](https://github.com/vllm-project/vllm/issues/63414) | [Bug] compile_mm_encoder 对 RADIO/NaViT encoder 致命崩溃 | 第一张图片请求即 EngineDeadError | 0 |
| [#63187](https://github.com/vllm-project/vllm/issues/63187) | [Bug] 多模态 pruning: encoder 计算预算被 placeholder count 扣减 → CUDA OOM | 2 |
| [#62969](https://github.com/vllm-project/vllm/issues/62969) | [Bug] Video pruning (EVS): M-RoPE positions 在 chunk 内部错误 | 2 |
| [#62968](https://github.com/vllm-project/vllm/issues/62968) | [Bug][PaliGemma] mm_prefix 双向 range 只覆盖 image tokens 不含 instruction | 0 |
| [#62937](https://github.com/vllm-project/vllm/issues/62937) | [Bug] Qwen3-VL video pruning: M-RoPE recompute 提前 num_timestamp_tokens+1 | 0 |
| [#62705](https://github.com/vllm-project/vllm/issues/62705) | [Bug] DiffusionGemma 对 image tokens 使用 causal attention | config 要求 bidirectional | 1 |

#### 分析与解决方向

- **M-RoPE 位置计算 ([#62969](https://github.com/vllm-project/vllm/issues/62969), [#62937](https://github.com/vllm-project/vllm/issues/62937))**：video pruning 时 M-RoPE 的位置重计算在 chunk 边界处出错——当 placeholder 的外层 `<|vision_start|>` 标签被保留时，recompute 起始位置偏移了 `num_timestamp_tokens + 1`。需要修复 `recompute_mrope_positions()` 的 chunk 边界逻辑。
- **Encoder 编译 ([#63414](https://github.com/vllm-project/vllm/issues/63414))**：`compile_mm_encoder=true` 对 RADIO/NaViT encoder 在首次图片请求时崩溃，`NoneType` attribute error。需要处理编译路径中 encoder 无 `mm_encoder` 属性的情况。
- **Pruning 计算预算 ([#63187](https://github.com/vllm-project/vllm/issues/63187))**：多模态 pruning 时 encoder 的计算预算被已 pruned 的 placeholder count 扣减，导致单步发送数倍于 profiled 的 patches 到 ViT，引发 CUDA OOM。需要改为按实际 patch 数而非 placeholder 数扣减。
- **Bidirectional attention ([#62968](https://github.com/vllm-project/vllm/issues/62968), [#62705](https://github.com/vllm-project/vllm/issues/62705))**：PaliGemma 的 prefix 应包含 image + instruction（整体 bidirectional），但 vLLM 只对 image tokens 建立 bidirectional range；DiffusionGemma 的 image token 应 bidirectional 但被 causal 处理。需要将 bidirectional range 扩展到完整 prefix。

---

### 2.8 ROCm / AMD GPU — 6 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63340](https://github.com/vllm-project/vllm/issues/63340) | (见 DSv4.1) Mono decode enable MI350P (128 CUs) | 2 |
| [#63336](https://github.com/vllm-project/vllm/issues/63336) | (见 KV Cache) MoRIIO transfer contract safe KV reclamation | 1 |
| [#63049](https://github.com/vllm-project/vllm/issues/63049) | [Bug] TritonW4A16 GEMM 在 M×N 或 M×K > 2^31 时 fault | int32 row offset 溢出 | 1 |
| [#62828](https://github.com/vllm-project/vllm/issues/62828) | [Bug] ROCM_ATTN sliding-window kernel 对 NaN/Inf KV slot 输出 NaN | 0 |
| [#62697](https://github.com/vllm-project/vllm/issues/62697) | [Bug] ROCM_ATTN 在 RDNA head size ≥ 96 + block ≠ 16 返回错误 attention | 可 fault GPU | 2 |
| [#62844](https://github.com/vllm-project/vllm/issues/62844) | [Installation] Gemma4 多模态在 ROCm 0.32.0 因视频编码器内存预留过多无法启动 | 2 |

#### 分析与解决方向

- **Triton int32 溢出 ([#63049](https://github.com/vllm-project/vllm/issues/63049))**：`triton_w4a16_gemm_kernel` 的 `offs_m = pid_m * BLOCK_M + tl.arange(...)` 在 int32 下当 M×K 或 M×N 超过 2^31 时溢出。修复：将 offset 计算改为 int64。
- **ROCM_ATTN NaN 传播 ([#62828](https://github.com/vllm-project/vllm/issues/62828))**：sliding-window 层中，window 外的 KV slot 被 mask（softmax weight=0）但仍然被加载，如果该 slot 含 NaN/Inf 会污染 softmax 计算。修复：对 masked slot 在加载后用 0 覆盖。
- **RDNA 兼容性 ([#62697](https://github.com/vllm-project/vllm/issues/62697))**：RDNA4 (gfx1201) 上 HIP paged-attention kernel 只支持 head 128 / block 16 / GQA 3-16，其余走 Triton fallback，但 fallback 在 head size ≥ 96 + block ≠ 16 时返回错误结果或 fault。需要扩展 HIP kernel 支持的 layout 范围或修复 Triton fallback。

---

### 2.9 性能优化 — 8 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63399](https://github.com/vllm-project/vllm/issues/63399) | [Performance] FlashInfer workspace 从 394→32 MiB，KV cache +21% | 4 并发 16K input，batch 完成时间 -5~6% | 1 |
| [#63198](https://github.com/vllm-project/vllm/issues/63198) | [Performance] SM100 prefill indexer: BF16 logits → FP32 top-k 多一次 pass | BF16-native top-k 快 20-25% | 0 |
| [#62919](https://github.com/vllm-project/vllm/issues/62919) | [Performance] Mamba2 continued prefill 密集 initial-state 材料化 | 可用 indexed gather 替代 | 0 |
| [#62768](https://github.com/vllm-project/vllm/issues/62768) | [Performance] RoutedExperts.load_weights 每次调用重建 expert mapping | runai_streamer 下 ~95k 次重建，50s | 4 |
| [#62862](https://github.com/vllm-project/vllm/issues/62862) | (见 KV Cache) MultiConnector 禁用 divergent local hybrid cache hits | 0 |
| [#62752](https://github.com/vllm-project/vllm/issues/62752) | [Performance] hybrid Attention/Mamba decode block 转换时不必要 preemption | 1 |
| [#62689](https://github.com/vllm-project/vllm/issues/62689) | [RFC] Layered prefill: split prefill across decoder layer groups | 替代 chunked prefill 降低 MoE expert 重载 | 0 |
| [#63006](https://github.com/vllm-project/vllm/issues/63006) | (见 KV Cache) Mooncake KV transfer bandwidth 优化 | 4 |

#### 分析与解决方向

- **FlashInfer workspace sizing ([#63399](https://github.com/vllm-project/vllm/issues/63399))**：启动时将 workspace 从默认 394 MiB 缩减到 32 MiB 可回收 21% KV cache 容量。建议根据 GPU 显存动态计算 workspace 大小。
- **BF16-native top-k ([#63198](https://github.com/vllm-project/vllm/issues/63198))**：DeepGEMM 返回 BF16 logits 后，prefill indexer 先 upcast 到 FP32 再做 top-k，浪费了 DeepGEMM 的加速。实现直接读取 BF16 的 `top_k_per_row_prefill` 可快 20-25%。
- **Expert mapping 缓存 ([#62768](https://github.com/vllm-project/vllm/issues/62768))**：`RoutedExperts.load_weights` 每次调用都重建 expert mapping，在 `runai_streamer` 按张量交错到达时产生 ~95k 次重建。修复：缓存 mapping 和 index，仅在首次调用时构建。
- **Layered prefill ([#62689](https://github.com/vllm-project/vllm/issues/62689))**：传统 chunked prefill 按 token 维度切分，每个 chunk 都要重载所有层的 expert weights。Layered prefill 按 layer 维度切分，一次推进一大批 token 通过一组 layer，减少 expert weight 重载次数。这是对 MoE 模型 prefill 吞吐的架构级优化。
- **Mamba2 initial state ([#62919](https://github.com/vllm-project/vllm/issues/62919))**：continued prefill 时 `torch.where` 材料化整个 dense state 张量，可用 indexed gather + scatter 避免全量临时分配。

---

### 2.10 LoRA — 3 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63285](https://github.com/vllm-project/vllm/issues/63285) | [Bug] SEQ_CLS adapters (modules_to_save=null) 加载崩溃 | classifier.weight 被误解析为 LoRA B 条目 | 0 |
| [#63278](https://github.com/vllm-project/vllm/issues/63278) | [Bug] BERT seq classification 丢弃 bert.pooler.dense 上的 LoRA | [#61279](https://github.com/vllm-project/vllm/issues/61279) 后 pooler 保持 nn.Linear | 0 |
| [#62801](https://github.com/vllm-project/vllm/issues/62801) | [Bug] MoE LoRA 接受不兼容的 round-robin expert placement + EP | LoRA 加载假设 contiguous ownership | 0 |

#### 分析与解决方向

- **SEQ_CLS adapter 解析 ([#63285](https://github.com/vllm-project/vllm/issues/63285))**：PEFT 的 `SEQ_CLS` task type 总是将 `classifier` 加入 `modules_to_save`，但 vLLM 不读 `task_type`，导致 `classifier.weight` 被误解析为 LoRA B 条目（无对应 A）。修复：在 LoRA name 解析时检查 `task_type` 并正确处理 `modules_to_save` 字段。
- **Pooler LoRA 丢失 ([#63278](https://github.com/vllm-project/vllm/issues/63278))**：`all-linear` target_modules 训练的 adapter 会在 `bert.pooler.dense` 上放 LoRA，但 vLLM 将该层保持为 `nn.Linear` 不支持 LoRA。需要将 pooler 层也纳入 LoRA 支持范围。
- **MoE LoRA + EP ([#62801](https://github.com/vllm-project/vllm/issues/62801))**：round-robin expert placement 与 LoRA 的 contiguous ownership 假设不兼容。需要在启动时检测不兼容的 placement 并拒绝或自动转换。

---

### 2.11 新硬件支持 (SM120 / GB10 / Jetson / Ascend) — 5 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63343](https://github.com/vllm-project/vllm/issues/63343) | (见 DSv4.1) SM120 adaptive verification 被拒绝 | 4 |
| [#63267](https://github.com/vllm-project/vllm/issues/63267) | [Bug] Jetson Thor (sm_110) 无 FA extensions 启动崩溃 | vllm_flash_attn import 时报错 | 1 |
| [#62984](https://github.com/vllm-project/vllm/issues/62984) | [Bug] b12x 1.5.1 paged-priming compile 崩溃 engine-core | unguarded fake_mode | 1 |
| [#63011](https://github.com/vllm-project/vllm/issues/63011) | [Bug] b12x attention backend head_dim 64 全部崩溃 | 无有效 NUM_MMA_KV typed shared-memory 分配 | 0 |
| [#63003](https://github.com/vllm-project/vllm/issues/63003) | [Bug] MiniMax-M3 ViT mRoPE on Ascend A5 UB 容量耗尽 | 0 |

#### 分析与解决方向

- **Jetson / sm_110 ([#63267](https://github.com/vllm-project/vllm/issues/63267))**：无 FA extensions 的平台上 `vllm_flash_attn` 在 import 时就崩溃，但 FA 仍被自动选为 attention backend。修复：(1) import 时 graceful degradation；(2) 在无 FA extensions 时自动 fallback 到其他 backend。
- **b12x backend ([#62984](https://github.com/vllm-project/vllm/issues/62984), [#63011](https://github.com/vllm-project/vllm/issues/63011))**：b12x 1.5.1 在 paged attention + chunked prefill 下 `fake_mode` 未保护导致编译崩溃；head_dim 64 无有效 shared-memory 分配。需要 b12x 上游修复 + vLLM 侧增加 fallback 逻辑。
- **Ascend A5 ([#63003](https://github.com/vllm-project/vllm/issues/63003))**：MiniMax-M3 的 ViT mRoPE 在 BLOCK_N=32 时超出 Unified Buffer 容量。需在 vllm-ascend 插件侧调整 block size。

---

### 2.12 安装 / 兼容性 — 7 个

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63121](https://github.com/vllm-project/vllm/issues/63121) | [Bug] torchcodec 未 pin 版本，cu129 镜像拉到 CUDA-13 wheel | vllm serve 无法启动 | 2 |
| [#62842](https://github.com/vllm-project/vllm/issues/62842) | [Installation] cu129 nightly 安装困难 | 总是拉到 cu13 库 | 0 |
| [#62844](https://github.com/vllm-project/vllm/issues/62844) | (见 ROCm) Gemma4 多模态 ROCm 0.32.0 内存预留过多 | 2 |
| [#62848](https://github.com/vllm-project/vllm/issues/62848) | [Bug] DeepSeek-OCR Triton CompilationError: `LOG2E` | v0.32.0 | 0 |
| [#62677](https://github.com/vllm-project/vllm/issues/62677) | (见量化) v0.32.0 deepep_v2 ZeroDivisionError | 0 |
| [#63020](https://github.com/vllm-project/vllm/issues/63020) | (见 KV Cache) flint NVFP4 GGUF serving 需 workarounds | 2 |
| [#63019](https://github.com/vllm-project/vllm/issues/63019) | [Bug] "Loading weights took" 日志显示 raw clock 而非耗时 | 197518 秒 | 1 |

#### 分析与解决方向

- **torchcodec 版本 pin ([#63121](https://github.com/vllm-project/vllm/issues/63121))**：cu129 镜像中 torchcodec 未 pin 版本，pip 解析到 CUDA-13 的 wheel，链接 `libnvrtc.so.13` 在 CUDA-12.9 环境下崩溃。修复：在 `requirements/build/cuda.txt` 或 image build 时固定 torchcodec 版本到 cu129 变体。
- **Triton LOG2E ([#62848](https://github.com/vllm-project/vllm/issues/62848))**：DeepEncoder 的 rel-pos attention kernel 使用了 `math.LOG2E` 全局变量，Triton jit 函数无法访问全局变量。修复：将 `LOG2E` 改为硬编码常量或 `tl.constexpr`。
- **日志 bug ([#63019](https://github.com/vllm-project/vllm/issues/63019))**：`DefaultLoader` 不从 weights iterator 拉取时，计时器记录的是 raw clock（~主机 uptime）而非实际加载耗时。修复：正确记录 start/end 时间差。

---

### 2.13 其他 — RFC / 文档 / CPU 后端等

| # | Issue | 问题摘要 | 评论 |
|---|-------|---------|:----:|
| [#63140](https://github.com/vllm-project/vllm/issues/63140) | [RFC] Stream request-level prefill progress over gRPC | 让 Router 的 kv_aware 策略用 worker 上报的 prefill 进度 | 0 |
| [#62815](https://github.com/vllm-project/vllm/issues/62815) | [Feature] Qwen4Exp PLE cpu_offload without UVA | NVIDIA Confidential Computing 下 UVA 不可用 | 0 |
| [#63057](https://github.com/vllm-project/vllm/issues/63057) | [Feature] Worker.sleep device-wide memory check 应 warn 而非 assert | 多实例共享 GPU 时 false positive | 0 |
| [#63164](https://github.com/vllm-project/vllm/issues/63164) | [Feature] Cohere grounding citations on derender paths | derender 路径丢失 grounding citations | 0 |
| [#63162](https://github.com/vllm-project/vllm/issues/63162) | [Bug] `return_tokens_as_token_ids` 在 derender 路径被忽略 | 2 |
| [#62776](https://github.com/vllm-project/vllm/issues/62776) | [Bug] Qwen3.8-Flash-Next GB10: 超过 indexer_budget (2048) 的 prompt 不可见 | 8 |
| [#62687](https://github.com/vllm-project/vllm/issues/62687) | [Bug] CPU backend: seeded request + chunked prefill 崩溃 | `torch.Generator` 不支持 `get_offset()` | 1 |
| [#62645](https://github.com/vllm-project/vllm/issues/62645) | [Bug] 加载另一个 config alias 改变已加载 config 的 model_type | 2 |
| [#62640](https://github.com/vllm-project/vllm/issues/62640) | [RFC] Improve MTP configuration warning | 当前警告过于激进且无具体建议 | 0 |

#### 分析与解决方向

- **indexer_budget 截断 ([#62776](https://github.com/vllm-project/vllm/issues/62776))**：Qwen3.8-Flash-Next 的 indexer 默认 budget 2048 token，超出部分对模型"不可见"。这是架构级问题——indexer 只看前 2K token 决定 top-k 路由，但需要确认是否应为长上下文场景增大 budget 或实现分层索引。
- **CPU 后端 seed + chunked prefill ([#62687](https://github.com/vllm-project/vllm/issues/62687))**：`torch.Generator.get_offset()` 在 CPU 上不支持，chunked prefill 切分带 seed 的 prompt 时崩溃。修复：在 chunked prefill 下禁用 seed 或改用确定性采样。
- **Config alias 污染 ([#62645](https://github.com/vllm-project/vllm/issues/62645))**：加载新的 config alias 会修改已加载 config 的序列化 `model_type`。这是一个全局状态污染 bug，需要在 config 加载时做深拷贝隔离。

---

## 三、关键趋势与建议

### 3.1 趋势总结

1. **DeepSeek-V4.1 是当前最大痛点来源**：10 个 issue 直接相关，涉及输出正确性、新硬件适配、推测解码集成、性能优化等多个维度。V4.1 的 piecewise CUDA graph 路径有系统性缺陷。

2. **推测解码生态快速扩张但稳定性不足**：DSpark、Eagle3/4、MTP、Dynamic SD 等多种方案并存，但 PP 兼容性、adaptive verification、graph padding 等方面都有未解决的 bug。

3. **API 兼容性债务累积**：Responses API 大量字段未实现或实现不完整；streaming vs non-streaming 路径逻辑分叉；Anthropic API 的 prefix cache 和 prefill 语义未正确实现。

4. **新硬件适配滞后**：SM120 (Blackwell)、GB10、Jetson Thor、MI350P、Ascend A5 等新硬件/新架构都有适配问题，attention backend 的自动选择逻辑需要更智能的 fallback。

5. **KV 连接器生态复杂化**：Mooncake、NIXL、LMCache、MoRIIO 等多种 connector 并存，但生命周期管理、事件正确性、安全释放等基础合约尚未统一。

### 3.2 建议优先级

| 优先级 | 方向 | 关键 Issues |
|:------:|------|------------|
| P0 | DeepSeek-V4.1 piecewise CUDA graph 输出正确性 | [#62811](https://github.com/vllm-project/vllm/issues/62811), [#63276](https://github.com/vllm-project/vllm/issues/63276), [#62935](https://github.com/vllm-project/vllm/issues/62935) |
| P0 | FULL graph + FA3 split-KV logits 错误 | [#62626](https://github.com/vllm-project/vllm/issues/62626) |
| P0 | torchcodec 版本 pin (cu129 镜像无法启动) | [#63121](https://github.com/vllm-project/vllm/issues/63121) |
| P1 | Streaming vs non-streaming tool calling 一致性 | [#63346](https://github.com/vllm-project/vllm/issues/63346), [#63358](https://github.com/vllm-project/vllm/issues/63358), [#63309](https://github.com/vllm-project/vllm/issues/63309) |
| P1 | KV connector 生命周期与安全释放 | [#62808](https://github.com/vllm-project/vllm/issues/62808), [#63336](https://github.com/vllm-project/vllm/issues/63336), [#63149](https://github.com/vllm-project/vllm/issues/63149) |
| P1 | MoE LoRA + EP expert placement 兼容性 | [#62801](https://github.com/vllm-project/vllm/issues/62801) |
| P1 | Qwen3.8-Flash-Next indexer_budget 截断 | [#62776](https://github.com/vllm-project/vllm/issues/62776) |
| P2 | FlashInfer q_scale / NVFP4 autotune 非确定性 | [#63043](https://github.com/vllm-project/vllm/issues/63043), [#62820](https://github.com/vllm-project/vllm/issues/62820) |
| P2 | M-RoPE video pruning 位置计算 | [#62969](https://github.com/vllm-project/vllm/issues/62969), [#62937](https://github.com/vllm-project/vllm/issues/62937) |
| P2 | Layered prefill RFC (MoE 吞吐优化) | [#62689](https://github.com/vllm-project/vllm/issues/62689) |
| P3 | Responses API 字段补全 | [#63183](https://github.com/vllm-project/vllm/issues/63183), [#63184](https://github.com/vllm-project/vllm/issues/63184), [#63185](https://github.com/vllm-project/vllm/issues/63185) |
| P3 | EAGLE-4 / model-based drafter 支持 | [#63004](https://github.com/vllm-project/vllm/issues/63004), [#62694](https://github.com/vllm-project/vllm/issues/62694) |

---

## 四、完整 Issue 列表

| # | 标题 | 创建日期 | 评论 | 标签 |
|---|------|:--------:|:----:|------|
| [#63439](https://github.com/vllm-project/vllm/issues/63439) | [RFC]: Reduce CPU overhead in Model Runner V2 for long-context P/D serving | 2026-10-24 | 1 | |
| [#63414](https://github.com/vllm-project/vllm/issues/63414) | [Bug]: compile_mm_encoder is engine-fatal for RADIO/NaViT encoders | 2026-10-23 | 0 | |
| [#63399](https://github.com/vllm-project/vllm/issues/63399) | [Performance]: Startup sizing of FlashInfer workspace to recover KV-cache capacity | 2026-10-23 | 1 | performance |
| [#63363](https://github.com/vllm-project/vllm/issues/63363) | [RFC]: Keep per-request token lists out of EngineCore garbage collection | 2026-10-23 | 0 | |
| [#63359](https://github.com/vllm-project/vllm/issues/63359) | [RFC]: Use session Agent Hints to reuse prompt tokens and block hashes | 2026-10-23 | 0 | RFC |
| [#63358](https://github.com/vllm-project/vllm/issues/63358) | [Bug]: Hermes tool parser: streaming and non-streaming disagree | 2026-10-23 | 0 | bug,tool-calling |
| [#63356](https://github.com/vllm-project/vllm/issues/63356) | [Bug]: Chat derender ignores `--response-role`, `add_generation_prompt: false` and `echo` | 2026-10-23 | 0 | bug |
| [#63346](https://github.com/vllm-project/vllm/issues/63346) | [Bug]: Streaming chat keeps tool calls for `tool_choice: "none"` | 2026-10-23 | 1 | bug,tool-calling,gpt-oss |
| [#63343](https://github.com/vllm-project/vllm/issues/63343) | [Bug][DSv4.1][SM120] enable_adaptive_verification is rejected on SM120 | 2026-10-23 | 4 | DSv4.1 |
| [#63340](https://github.com/vllm-project/vllm/issues/63340) | [Feature][ROCm][DSv4.1] Mono decode: enable MI350P (gfx950, 128 CUs) | 2026-10-23 | 2 | rocm,DSv4.1 |
| [#63336](https://github.com/vllm-project/vllm/issues/63336) | [RFC]: MoRIIO transfer contract: safe KV reclamation and bounded failure for P/D | 2026-10-23 | 1 | rocm,RFC |
| [#63331](https://github.com/vllm-project/vllm/issues/63331) | Anthropic /v1/messages: inline system messages merged → prefix cache miss | 2026-10-23 | 1 | |
| [#63310](https://github.com/vllm-project/vllm/issues/63310) | [Bug]: Responses API rejects assistant input items without "type" | 2026-10-23 | 0 | bug |
| [#63309](https://github.com/vllm-project/vllm/issues/63309) | [Bug]: Hermes tool parser streaming silently drops content after `<<` marker | 2026-10-23 | 1 | bug,tool-calling |
| [#63285](https://github.com/vllm-project/vllm/issues/63285) | [Bug][LoRA] SEQ_CLS adapters saved with modules_to_save=null crash on load | 2026-10-23 | 0 | |
| [#63278](https://github.com/vllm-project/vllm/issues/63278) | [Bug][LoRA] BERT sequence classification drops LoRA on bert.pooler.dense | 2026-10-23 | 0 | pooling |
| [#63276](https://github.com/vllm-project/vllm/issues/63276) | [Bug]: DS V4.1 Flash1017 outputs incorrect numbers | 2026-10-23 | 1 | bug |
| [#63267](https://github.com/vllm-project/vllm/issues/63267) | [Bug]: CUDA builds without FA extensions (Jetson Thor sm_110) cannot start | 2026-10-23 | 1 | |
| [#63250](https://github.com/vllm-project/vllm/issues/63250) | [Bug]: DeepSeek-V4.1-Flash(-1017) returns U+FFFD for a 12-token prompt | 2026-10-23 | 1 | deepseek,DSv4.1 |
| [#63240](https://github.com/vllm-project/vllm/issues/63240) | [Question]: Qwen4Exp PLE n-gram table is materialised dense in bf16 | 2026-10-23 | 1 | quantization |
| [#63215](https://github.com/vllm-project/vllm/issues/63215) | [RFC]: Declared ownership and layout for WeightSource | 2026-10-22 | 3 | RFC |
| [#63198](https://github.com/vllm-project/vllm/issues/63198) | [Performance]: SM100 prefill indexer: BF16 logits upcast to FP32 before top-k | 2026-10-22 | 0 | |
| [#63187](https://github.com/vllm-project/vllm/issues/63187) | [Bug]: Multimodal pruning: encoder compute budget debited by pruned placeholder count → OOM | 2026-10-22 | 2 | bug,multi-modality |
| [#63185](https://github.com/vllm-project/vllm/issues/63185) | [Bug]: Responses API rejects `include: ["web_search_call.*"]` with 400 | 2026-10-22 | 0 | tool-calling |
| [#63184](https://github.com/vllm-project/vllm/issues/63184) | [Bug]: Responses API: `reasoning.summary` always returns `[]` | 2026-10-22 | 1 | tool-calling |
| [#63183](https://github.com/vllm-project/vllm/issues/63183) | [Bug]: Responses API: completed responses never carry `completed_at` | 2026-10-22 | 0 | |
| [#63175](https://github.com/vllm-project/vllm/issues/63175) | [Bug]: Triton ReplaySSM (V1) drops un-flushed ring tokens when request skips a step | 2026-10-22 | 0 | |
| [#63170](https://github.com/vllm-project/vllm/issues/63170) | [Bug]: GLM-5.x (DSA) under PP: reuse layers read stale top-k indices | 2026-10-22 | 4 | bug,glm |
| [#63168](https://github.com/vllm-project/vllm/issues/63168) | [Bug]: GLM-5.3-Flash kpool prefill writer leaves pool straddling unaligned chunk | 2026-10-22 | 0 | glm |
| [#63164](https://github.com/vllm-project/vllm/issues/63164) | [Feature]: Cohere grounding citations on the derender paths | 2026-10-22 | 0 | feature request |
| [#63162](https://github.com/vllm-project/vllm/issues/63162) | [Bug]: `return_tokens_as_token_ids` is ignored on derender paths | 2026-10-22 | 2 | bug |
| [#63157](https://github.com/vllm-project/vllm/issues/63157) | [Spec Decode][DeepSeek-V4.1] Support DSpark with pipeline parallelism | 2026-10-22 | 1 | deepseek,DSv4.1 |
| [#63152](https://github.com/vllm-project/vllm/issues/63152) | [Tracking] Dory looped model: per-request loops, spec decode, GB10 performance | 2026-10-22 | 0 | speculative-decoding,quantization |
| [#63149](https://github.com/vllm-project/vllm/issues/63149) | [Tracking] KV event correctness and compatibility | 2026-10-22 | 8 | |
| [#63140](https://github.com/vllm-project/vllm/issues/63140) | [RFC] Stream request-level prefill progress over gRPC | 2026-10-22 | 0 | |
| [#63130](https://github.com/vllm-project/vllm/issues/63130) | [Bug]: Strict serde rejection of duplicate field `reasoning` breaks agent harnesses | 2026-10-22 | 3 | bug,tool-calling |
| [#63128](https://github.com/vllm-project/vllm/issues/63128) | [Bug][Spec Decode] MTP draft runs with uninitialised embedding under PP | 2026-10-22 | 1 | quantization |
| [#63121](https://github.com/vllm-project/vllm/issues/63121) | [Bug]: `torchcodec` is unpinned, resolves to CUDA-13 wheel in cu129 builds | 2026-10-22 | 2 | bug |
| [#63058](https://github.com/vllm-project/vllm/issues/63058) | [Feature]: keep_tags for CuMemAllocator.sleep | 2026-10-22 | 2 | |
| [#63057](https://github.com/vllm-project/vllm/issues/63057) | [Feature]: Worker.sleep device-wide memory check should warn, not assert | 2026-10-22 | 0 | |
| [#63052](https://github.com/vllm-project/vllm/issues/63052) | [Bug]: Breakable PIECEWISE capture leaves warmup activations cached | 2026-10-22 | 0 | bug |
| [#63049](https://github.com/vllm-project/vllm/issues/63049) | [Bug]: TritonW4A16 GEMM faults on ROCm once M × N or M × K > 2^31 | 2026-10-22 | 1 | rocm |
| [#63043](https://github.com/vllm-project/vllm/issues/63043) | [Bug]: Native FlashInfer produces garbage output with BF16 Q and non-unit q_scale | 2026-10-22 | 0 | quantization |
| [#63037](https://github.com/vllm-project/vllm/issues/63037) | [Bug][KV Offload] Reused OffloadKey can lose metadata from delayed BlockStored events | 2026-10-22 | 0 | |
| [#63020](https://github.com/vllm-project/vllm/issues/63020) | [Bug]: vllm-openai:flint can't serve NVFP4 GGUF checkpoint without workarounds | 2026-10-22 | 2 | |
| [#63019](https://github.com/vllm-project/vllm/issues/63019) | [Bug]: "Loading weights took" logs raw clock (~host uptime) | 2026-10-22 | 1 | |
| [#63011](https://github.com/vllm-project/vllm/issues/63011) | [Bug]: b12x attention backend advertises head_dim 64 but crashes at first forward | 2026-10-22 | 0 | |
| [#63006](https://github.com/vllm-project/vllm/issues/63006) | [RFC]: Reduce descriptor and WR overhead to improve Mooncake KV transfer bandwidth | 2026-10-22 | 4 | RFC,kv-connector |
| [#63004](https://github.com/vllm-project/vllm/issues/63004) | [RFC]: add eagle4 spec decode | 2026-10-22 | 1 | RFC |
| [#63003](https://github.com/vllm-project/vllm/issues/63003) | [Bug]: MiniMax-M3 ViT mRoPE setup BLOCK_N=32 exceeds UB capacity on Ascend A5 | 2026-10-22 | 0 | minimax |
| [#63000](https://github.com/vllm-project/vllm/issues/63000) | Question: long-context generation stops mid-sentence with `<\|im_end\|>` on Qwen3.8-27B | 2026-10-21 | 1 | quantization |
| [#62984](https://github.com/vllm-project/vllm/issues/62984) | [Bug]: [b12x] extra pins b12x 1.5.1 — unguarded `fake_mode` crashes engine-core | 2026-10-21 | 1 | |
| [#62969](https://github.com/vllm-project/vllm/issues/62969) | [Bug]: Video pruning (EVS): M-RoPE positions are wrong for chunks inside video placeholder | 2026-10-21 | 2 | |
| [#62968](https://github.com/vllm-project/vllm/issues/62968) | [Bug][PaliGemma] mm_prefix bidirectional ranges cover only image tokens | 2026-10-21 | 0 | |
| [#62965](https://github.com/vllm-project/vllm/issues/62965) | [Feature]: Capture DSv4.1 decoder replay graphs on model graphs' stream | 2026-10-21 | 0 | feature request,deepseek,DSv4.1 |
| [#62945](https://github.com/vllm-project/vllm/issues/62945) | [Bug]: nvidia/GLM-5.3-NVFP4: silent garbage output with EP + TP>1 + PCP>1 | 2026-10-21 | 0 | bug,nvidia,glm |
| [#62939](https://github.com/vllm-project/vllm/issues/62939) | [Bug][MRV2][Spec Decode] DSpark + adaptive verification fails at startup | 2026-10-21 | 3 | |
| [#62937](https://github.com/vllm-project/vllm/issues/62937) | [Bug]: Qwen3-VL video pruning: M-RoPE recompute starts too early | 2026-10-21 | 0 | bug |
| [#62935](https://github.com/vllm-project/vllm/issues/62935) | [Bug]: DeepSeek-V4.1-Flash-1017 on GB300: long agent replies fall into repetition loops | 2026-10-21 | 1 | deepseek,quantization,DSv4.1 |
| [#62931](https://github.com/vllm-project/vllm/issues/62931) | [Bug]: Tiered KV offload: secondary-tier hit larger than CPU tier loads nothing | 2026-10-21 | 0 | |
| [#62919](https://github.com/vllm-project/vllm/issues/62919) | [Performance]: Avoid dense initial-state materialization in Mamba2 continued prefill | 2026-10-21 | 0 | performance |
| [#62866](https://github.com/vllm-project/vllm/issues/62866) | [Doc]: MooncakeStoreConnector guide does not mention kv_load_failure_policy | 2026-10-21 | 0 | |
| [#62865](https://github.com/vllm-project/vllm/issues/62865) | [Bug]: Compile mode 2 leads to incorrect kernels with Transformers backend | 2026-10-21 | 0 | bug,kimi |
| [#62862](https://github.com/vllm-project/vllm/issues/62862) | [Performance]: MultiConnector disables divergent local hybrid cache hits on Decode | 2026-10-21 | 0 | |
| [#62851](https://github.com/vllm-project/vllm/issues/62851) | [Bug]: Qwen3.6 GDN greedy DFlash2 divergence from batch-dependent numerics | 2026-10-21 | 3 | quantization |
| [#62848](https://github.com/vllm-project/vllm/issues/62848) | [Bug]: DeepSeek-OCR / Unlimited-OCR fail to start — Triton CompilationError LOG2E | 2026-10-21 | 0 | bug,deepseek |
| [#62844](https://github.com/vllm-project/vllm/issues/62844) | [Installation]: [ROCm] Gemma4 multimodal fails to start due to excessive video encoder memory | 2026-10-21 | 2 | installation,rocm,multi-modality |
| [#62842](https://github.com/vllm-project/vllm/issues/62842) | [Installation]: cu129 | 2026-10-21 | 0 | installation |
| [#62839](https://github.com/vllm-project/vllm/issues/62839) | [RFC]: Fuse DSpark confidence-head GEMV into a single TileLang kernel | 2026-10-21 | 1 | |
| [#62830](https://github.com/vllm-project/vllm/issues/62830) | [Bug][Anthropic API] /v1/messages ignores assistant prefill | 2026-10-21 | 1 | |
| [#62828](https://github.com/vllm-project/vllm/issues/62828) | [Bug][ROCm]: ROCM_ATTN sliding-window kernels output NaN for out-of-window NaN/Inf KV | 2026-10-21 | 1 | rocm |
| [#62820](https://github.com/vllm-project/vllm/issues/62820) | [Bug]: FlashInfer fp4_gemm autotune changes temperature-0 tokens on Qwen3.8-27B NVFP4 | 2026-10-21 | 1 | quantization |
| [#62818](https://github.com/vllm-project/vllm/issues/62818) | [RFC]: Preserve output distribution under structured-output + speculative decoding | 2026-10-21 | 3 | structured-output,RFC,speculative-decoding |
| [#62815](https://github.com/vllm-project/vllm/issues/62815) | [Feature]: Qwen4Exp PLE cpu_offload without UVA (NVIDIA Confidential Computing) | 2026-10-21 | 0 | nvidia,quantization |
| [#62811](https://github.com/vllm-project/vllm/issues/62811) | [Bug]: DeepSeek-V4.1: wrong output when last chunk of chunked prefill runs as PIECEWISE graph | 2026-10-21 | 0 | deepseek,DSv4.1 |
| [#62810](https://github.com/vllm-project/vllm/issues/62810) | [RFC] Expose allocator-issued KV block generations to asynchronous connectors | 2026-10-21 | 0 | |
| [#62808](https://github.com/vllm-project/vllm/issues/62808) | [Bug]: LMCacheConnectorV1 does not forward shutdown(), EngineCore force-killed | 2026-10-21 | 2 | kv-connector |
| [#62803](https://github.com/vllm-project/vllm/issues/62803) | [Bug]: Investigate uniform decision scores for a D3 checkpoint loaded as Qwen3.5 | 2026-10-21 | 1 | bug,quantization |
| [#62801](https://github.com/vllm-project/vllm/issues/62801) | [Bug]: MoE LoRA accepts incompatible round-robin expert placement with EP | 2026-10-21 | 0 | |
| [#62793](https://github.com/vllm-project/vllm/issues/62793) | [Bug]: MRV2 adaptive verification reduces Eagle3 acceptance due to mismatched graph padding | 2026-10-21 | 0 | quantization |
| [#62781](https://github.com/vllm-project/vllm/issues/62781) | [Bug][XPU] GDN fp16 overflow reproduces on Intel XPU (Qwen3.5-family) | 2026-10-21 | 3 | intel-gpu |
| [#62776](https://github.com/vllm-project/vllm/issues/62776) | [Bug] Qwen3.8-Flash-Next on GB10: prompt beyond indexer_budget (2048) is invisible | 2026-10-20 | 8 | |
| [#62768](https://github.com/vllm-project/vllm/issues/62768) | [Performance]: RoutedExperts.load_weights rebuilds expert mapping on every call (~50s) | 2026-10-20 | 4 | |
| [#62762](https://github.com/vllm-project/vllm/issues/62762) | [Performance]: XPU block FP8: W8A16 instead of W8A8 took decode from 35 to 58 tok/s | 2026-10-20 | 1 | intel-gpu,quantization |
| [#62752](https://github.com/vllm-project/vllm/issues/62752) | [Performance]: Avoidable preemption at hybrid Attention/Mamba decode block transitions | 2026-10-20 | 1 | |
| [#62743](https://github.com/vllm-project/vllm/issues/62743) | [Bug]: K2 Horizon reasoning parser only accepts end tag of requested effort | 2026-10-20 | 0 | tool-calling,quantization |
| [#62740](https://github.com/vllm-project/vllm/issues/62740) | [Feature]: Measure the Dynamic SD table instead of writing it by hand | 2026-10-20 | 0 | |
| [#62724](https://github.com/vllm-project/vllm/issues/62724) | [Feature]: Token-aware control-token sanitization for DeepSeek renderers | 2026-10-20 | 3 | deepseek |
| [#62707](https://github.com/vllm-project/vllm/issues/62707) | [Bug]: v0.25.1 streaming input beyond max_model_len crashes V2 runner | 2026-10-20 | 1 | quantization |
| [#62705](https://github.com/vllm-project/vllm/issues/62705) | [Bug]: DiffusionGemma prefills image tokens causally, config asks for bidirectional | 2026-10-20 | 1 | |
| [#62697](https://github.com/vllm-project/vllm/issues/62697) | [Bug][ROCm] ROCM_ATTN returns wrong attention on RDNA for head size ≥ 96 | 2026-10-20 | 2 | rocm |
| [#62694](https://github.com/vllm-project/vllm/issues/62694) | [Feature]: [Spec Decode] Support model-based drafters (target hidden states) | 2026-10-20 | 0 | feature request |
| [#62689](https://github.com/vllm-project/vllm/issues/62689) | [RFC]: Layered prefill: split prefill across decoder layer groups | 2026-10-20 | 0 | |
| [#62687](https://github.com/vllm-project/vllm/issues/62687) | [Bug]: CPU backend: seeded request + chunked prefill kills engine | 2026-10-20 | 1 | cpu |
| [#62677](https://github.com/vllm-project/vllm/issues/62677) | [Bug]: v0.32.0 official image: deepep_v2 fails with ZeroDivisionError | 2026-10-20 | 0 | bug,quantization |
| [#62645](https://github.com/vllm-project/vllm/issues/62645) | [Bug]: Loading another config alias changes previously loaded config's model_type | 2026-10-20 | 2 | |
| [#62641](https://github.com/vllm-project/vllm/issues/62641) | [Usage]: Attempts to avoid full prompt processing when changing the tool set | 2026-10-20 | 0 | usage,tool-calling |
| [#62640](https://github.com/vllm-project/vllm/issues/62640) | [RFC]: Improve MTP configuration warning | 2026-10-20 | 0 | RFC |
| [#62639](https://github.com/vllm-project/vllm/issues/62639) | [Bug]: Responses API: tool_choice=allowed_tools does not work | 2026-10-20 | 3 | bug,tool-calling |
| [#62626](https://github.com/vllm-project/vllm/issues/62626) | [Bug]: `cudagraph_mode=FULL` + FA3 split-KV returns wrong logits | 2026-10-20 | 1 | bug,scheduler |
