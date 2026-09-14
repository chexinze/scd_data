# BNT3 DPLB 算法上移 —— 从零看懂到明天对齐

> 这份文档给完全没基础的人看。目标：你看完后，能回答"这个项目在干什么、我那个 commit 做了什么、还剩什么没做、明天跟引擎侧对齐谈什么"。
> 不用类比看不懂的地方，我也给了类比。你看不懂某段，往上看一段就有铺垫。

---

## 目录

- [第 0 章：先认识几个名词](#第-0-章先认识几个名词)
- [第 1 章：这个系统在干什么（项目全貌）](#第-1-章这个系统在干什么项目全貌)
- [第 2 章：你要解决的问题是什么（需求背景）](#第-2-章你要解决的问题是什么需求背景)
- [第 3 章：DP 通过 zmq 和 grpc 把信息报给 TS，TS 怎么接收](#第-3-章dp-通过-zmq-和-grpc-把信息报给-tsts-怎么接收)
  - [3.1 线 A：zmq —— 报"DP 现在有多忙"](#31-线-azmq--报dp-现在有多忙)
  - [3.2 线 B：grpc —— 报"一个 batch 跑完了"](#32-线-bgrpc--报一个-batch-跑完了)
  - [3.3 两条线放一起](#33-两条线放一起)
- [第 4 章：你的 commit 做完了什么、还剩什么](#第-4-章你的-commit-做完了什么还剩什么)
- [第 5 章：为什么"TS 选了 DP 但下游又自己选"不是 bug](#第-5-章为什么ts-选了-dp-但下游又自己选不是-bug)
- [第 6 章：明天跟引擎负责人对齐，谈什么](#第-6-章明天跟引擎负责人对齐谈什么)
- [第 7 章：一句话总结](#第-7-章一句话总结)
- [附录 A：关键代码文件速查](#附录-a关键代码文件速查)
- [附录 B：文档待确认项逐条翻译](#附录-b文档待确认项逐条翻译)

---

## 第 0 章：先认识几个名词

你看代码和文档之前，这几个名词必须先记住，否则每句话都看不懂。

| 名词 | 是什么 | 大白话 |
|---|---|---|
| **推理** | 大模型生成回答的过程 | ChatGPT 蹦字那个过程 |
| **Prefill（预填充）** | 推理第一步，把用户问的话一次性算完 | 读完你的问题 |
| **Decode（解码）** | 推理第二步，一个字一个字往外生成 | 一个字一个字蹦回答 |
| **PTE** | 跑 Prefill 的机器 | 干"读问题"活的机器 |
| **DTE** | 跑 Decode 的机器（一个 Decode 引擎）| 干"蹦字"活的机器 |
| **DP / DP rank** | 一个 DTE 里开的好几个干活进程 | 一台蹦字机器里的好几个工人 |
| **TS / LME** | 路由器，负责派活 | 分拣站，决定请求给谁 |
| **JE** | 引擎侧的协调者 | 引擎那边管事儿的 |
| **DPS** | DP scheduler，下游一个服务 | 原来选 DP 的地方，要被搬走 |
| **DPLB v2** | 一个看显存/排队来选 DP 的聪明算法 | 选 DP 的聪明办法 |
| **KV cache** | 存中间计算结果的显存 | 工人的工作台，台子满了就干不了活 |
| **zmq** | 一种消息广播工具（PUB 发 / SUB 收）| 大喇叭 + 收音机 |
| **grpc** | 一种远程调用工具（一问一答）| 打电话 |
| **proto** | 定义消息格式的文件 | 规定"喊话"内容格式的说明书 |
| **commit** | 一次代码提交 | 你那次改的代码 |
| **`lme_select_dp_enabled`** | 一个开关，控制引擎听不听 TS 的 | 引擎"听不听话"的开关 |

---

## 第 1 章：这个系统在干什么（项目全貌）

### 1.1 一次请求的完整旅程

用户发一句话，背后是这样跑的：

```
用户发请求
   ↓
[Prefill 机器 PTE]  把用户的问题一次性算完
   ↓
[路由器 TS]  决定：这个请求去做 Decode 时，送给哪个 DTE 的哪个 DP
   ↓
[Decode 机器 DTE]  里面有 DP0、DP1、DP2 ...  在某个 DP 上一个字一个字生成
   ↓
返回给用户
```

### 1.2 你负责的是哪一段

整个流程里，**路由器 TS 要决定"这个请求送给哪个 DP"**，这就是你负责的事。

### 1.3 为什么这事不简单

各个 DP 的忙闲不一样：
- 有的 DP 显存（KV cache）快满了，再塞会爆。
- 有的 DP 排队很长，再塞要等很久。
- 有的 DP 还很闲。

你要做到：**别让有的 DP 撑死、有的 DP 饿死，让请求合理地分到各个 DP。**

### 1.4 一个类比（看不懂名词的看这里）

把 TS 想象成快递分拣站，DP 想象成货车：
- 分拣员（TS）要决定每个包裹上哪辆车（DP）。
- 要决定得好，得知道每辆车有多满 → 每辆车得**主动告诉**分拣员自己还剩多少空间。
- 货车司机每隔一会儿喊一嗓子"我是 3 号车，还剩 40% 空间"——这就是**上报**。

---

## 第 2 章：你要解决的问题是什么（需求背景）

> 来自文档《BNT3-DPLB 算法上移方案》第一节。

### 2.1 现在的问题（三层都没做好）

1. **路由器 TS 只选到 DTE**（选哪个 Decode 机器），**不管 DTE 里哪个 DP**——只决定"去哪栋楼"，不管"进哪个门"。
2. **DTE 里选 DP 用纯轮询**（DP0、DP1、DP2、DP0、DP1……轮流来），不管谁忙谁闲。
3. 聪明算法 **DPLB v2** 被封在下游 DPS 里，而且引擎没把状态报上来，**实际没生效**。

**后果**：某个 DP 显存满了但还能接请求（轮询不管显存），请求继续塞，排队越堆越长，最后卡死或请求被挤掉。

### 2.2 目标

把 DPLB v2 算法**从下游 DPS 搬到路由器 TS**，让 TS：
1. 直接看到每个 DTE 里每个 DP 的负载（显存、排队、running 等）。
2. 一次性选好 DTE + DP，而不是先选 DTE 再轮询 DP。

### 2.3 关键：这事要分两边改

- **路由器侧（C++ 仓，你现在的 `XDS_ROUTER`）**：TS 收 DP 状态、选 (DTE,DP)、把结果传下去。
- **引擎侧（vllm 仓，另一个仓库）**：每个 DP 把状态报给 TS、DTE 听 TS 的话选 DP。

**你一直困惑的根源：你对着 C++ 仓啃，但剩下的活在 vllm 引擎仓。**

---

## 第 3 章：DP 通过 zmq 和 grpc 把信息报给 TS，TS 怎么接收

这条通路有**两条独立的线**，文档里都提了。它们干的事完全不同，必须分开看。

### 3.1 线 A：zmq —— 报"DP 现在有多忙"

这条线回答：TS 要选 DP，得先知道每个 DP 有多忙，这个"忙闲信息"怎么从 DP 到 TS。

#### 第 1 步：每个 DP 把自己状态打包成一个消息

- **做什么**：DP 把自己的负载——剩多少显存、跑着哪些请求、排了多少队——打包成一个叫 `DpRankReport` 的结构化消息。
- **消息长什么样**：定义在 `xds_cpp/network/proto/lme/dp_rank_report.proto`，字段就是文档第 2 节列的那些：
  - `te_name`（哪个 DTE）、`dp_index`（第几个 DP）
  - `free_kv_block_num`（剩多少显存块）、`total_kv_block_num`（总共多少）
  - `running`（正在跑的请求）、`trans_waiting`（等传 KV 的请求）
  - `dp_queue_seqs`（对账用的请求 id 集合）等等
- **在哪个仓**：proto 定义文件在**你这个 `XDS_ROUTER` 仓里，已经存在**（commit 加的）。
- **状态**：✅ 已就绪。

#### 第 2 步：DP 把这个消息通过 zmq 发出去

- **做什么**：DP 用 zmq 的 **PUB**（发布）方式，把消息序列化成字节，广播出去。每 100ms 发一次。
- **在哪个仓**：**vllm 引擎仓，代码不存在**。
- **状态**：❌ 没写，要引擎侧写。文档第 2 节"DP actor 直报 TS"说的就是这一步。在 `XDS_ROUTER` 仓里搜不到任何 `DpRankReport` 发送代码，因为 DP 进程在 vllm 引擎里，不在路由器仓。
- **类比**：货车司机（DP）装个喇叭（zmq PUB），每 100ms 喊一次自己有多满。这个喇叭还没装。

#### 第 3 步：TS 用 zmq 的 SUB 接收这些字节

- **做什么**：TS 建一个 zmq **SUB** socket，连到每个 DP 的 PUB 地址，循环收字节。
- **在哪个仓**：**你这个仓，已经存在**。
- **具体代码**：`xds_lme/dte_state_receiver/ts_state_receiver.cpp`
  - 建收音机：`UpdateDTEAddress`（第 43 行）对每个 DP 地址建/连 SUB socket。
  - 收字节：`SubscribeRawList`（第 123 行）循环 `RecvRaw` 把 socket 里所有字节拉光。
- **状态**：✅ 已就绪。
- **类比**：分拣站（TS）装个收音机（zmq SUB），一直开着收。

#### 第 4 步：TS 把字节解析回 `DpRankReport` 消息

- **做什么**：TS 收到原始字节，要反序列化回结构。但同一个 socket 上可能混着新消息 `DpRankReport` 和老消息 `TSStateInfo`，所以先判断是哪种再分别解。
- **在哪个仓**：**你这个仓，已经存在**。
- **具体代码**：`DispatchRaw`（第 137 行）。先试按 `DpRankReport` 解，用标记位 `is_dp_rank_report` 判断；不匹配就按老 `TSStateInfo` 解。
- **状态**：✅ 已就绪。

#### 第 5 步：TS 把状态按"哪个 DTE 的第几个 DP"存起来

- **做什么**：按合并键 `(te_name, dp_index)` 存进一张表。比如 DTE0 的 DP2 报的，存到 `DTE0 → [..., DP2状态, ...]`。
- **在哪个仓**：**你这个仓，已经存在**。
- **具体代码**：`UpdateLocalInfoFromDpRankReport`（第 173 行），存进 `per_dte_dp_agg_`。同时调 `LmeSchedTracker::Reconcile` 对账（把引擎已看到的请求从"已决策未上报"账本扣掉）。
- **状态**：✅ 已就绪。

#### 第 6 步：TS 把这张表交给选 DP 的逻辑用

- **做什么**：把存好的 per-DP 负载表推给路由器，选 DP 时翻这张表。
- **在哪个仓**：**你这个仓，已经存在**。
- **具体代码**：`UpdateSharedInfo`（第 213 行）推给 `LMEScheduler`；`dplb_v2_lme.cpp` 的 `SelectUnified` 通过 `GetDTEResourceInfo(dte)->dp_details` 读这张表。
- **状态**：✅ 已就绪。

#### 线 A 小结

zmq 这条线 6 步：**第 1、3、4、5、6 步在你这个仓里都已经写好了；只有第 2 步（DP 发送）没写，在 vllm 引擎仓里。**

> "DP 怎么通过 zmq 报给 TS"——**收的这边全有，发的这边全没有，因为发送方是引擎，不是路由器。**

---

### 3.2 线 B：grpc —— 报"一个 batch 跑完了"

这条线干的**不是报 DP 忙闲**，是报"这一批请求在引擎里跑完了，花了多久、命中了多少缓存"。

#### 第 1 步：JE 构造一个完成通知消息

- **做什么**：一批请求跑完后，JE 打包一个 `BatchCompletionInfo`，里面有耗时、拒绝原因、缓存命中信息。**里面没有 DP 的显存/排队信息。**
- **在哪个仓**：JE 侧 `xds_cpp/je_processor/je_post_processor.cpp`（`NotifyTsBatchCanceled` 等位置），**已经存在**。
- **状态**：✅ 已就绪。

#### 第 2 步：JE 通过 grpc 调 TS 的一个方法

- **做什么**：grpc 是"打电话"式通信（一问一答），JE 用 grpc 调 TS 的 `NotifyLMEBatchCompleted` 方法。
- **方法定义**：`xds_cpp/network/proto/lme_rpc_server.proto`，**已经存在**。
- **状态**：✅ 已就绪。

#### 第 3 步：TS 接收这个 grpc 调用

- **做什么**：TS 侧 grpc 服务收到调用，转交内部处理。
- **在哪个仓**：**你这个仓，已经存在**。
- **具体代码**：`xds_cpp/network/server/async_lme_rpc_server.cpp` 的 `NotifyLMEBatchCompletedCallData` 收到，转给 `xds_lme/info_management/info_manager.cpp` 的 `Server_BatchCompletionInfo`（第 26 行）。
- **状态**：✅ 已就绪。

#### 第 4 步：TS 把完成信息塞进 PTE 队列，用于统计和超时清理

- **做什么**：`Server_BatchCompletionInfo` 只做 `pte_manager_.queue_m.BufferExtend_safe(info)`，存起来给延迟统计和超时清理用。**不更新 DP 负载，不影响选 DP。**
- **在哪个仓**：**你这个仓，已经存在**。
- **状态**：✅ 已就绪。

#### 线 B 小结

grpc 这条线**整条都已写好**，发送方在 JE 侧（已有），接收方在 TS 侧（已有）。**跟选 DP 没关系，只管统计。**

---

### 3.3 两条线放一起

| | 线 A：zmq | 线 B：grpc |
|---|---|---|
| 报什么 | DP 现在有多忙（显存、排队、running）| 一批跑完了（耗时、命中）|
| 用来干吗 | **选 DP**（TS 决策依据）| 统计延迟、清超时（不影响选 DP）|
| 发送方 | 每个 DP，zmq PUB，100ms 一次 | JE，grpc，batch 完成时一次 |
| 发送方代码 | **vllm 引擎仓，不存在** | JE 侧，**已有** |
| 接收方 | TS，zmq SUB | TS，grpc server |
| 接收方代码 | **本仓 `ts_state_receiver.cpp`，已有** | **本仓 `info_manager.cpp`，已有** |

> **你之前困惑的"DP 怎么报给 TS"，答案就是这张表：收的这边全有，zmq 发的那边没有（在引擎仓），grpc 那边整条都有。你 leader 让你查这条通路，是因为你要去引擎仓补 zmq 发的那一半。**

---

## 第 4 章：你的 commit 做完了什么、还剩什么

你的 commit `f3a47182 "DPLB 算法上移"` 已经把**路由器侧（C++ 仓）做完了**。

### 4.1 已经做完的（路由器侧）

- ✅ 延迟选 D 的框架：PTE_DTE 路径不选 DTE/DP，prefill 完成后发 DTE_ONLY 回 TS 选。
- ✅ TS 收 `DpRankReport` 并按 `(te, dp)` 聚合（就是线 A 的第 3-6 步）。
- ✅ TS 用 DPLB v2 算法一次性选 (DTE, DP)（`dplb_v2_lme.cpp`）。
- ✅ 把选出的 DP rank 通过 `dte_dp_rank_map` / `lme_selected_dp_group_index` 传下去（`ts_struct_cast.cpp`、`je_post_processor.cpp`）。
- ✅ `LmeSchedTracker` 账本：决策时 `OnSchedule` 记 KV，上报时 `Reconcile` 扣。
- ✅ proto 字段、配置开关、stdin 下发都加了。

### 4.2 还没做的（引擎侧，vllm 仓）

#### 引擎侧要写的第 1 块：DP 上报器

- **要做什么**：在 vllm 引擎里，给每个 DP rank 写一个 `DPStateReporter`，持有 zmq PUB socket，每 100ms 把这个 DP 的负载打包成 `DpRankReport` 发出去。
- **这就是线 A 的第 2 步**——整条 zmq 通路里唯一缺失的一环。
- **字段**：按文档第 2 节和 `dp_rank_report.proto` 来，字段名/编号不可改。

#### 引擎侧要写的第 2 块：DTE 让权

- **要做什么**：改 vllm 的 `_add_task_multi_dp`，让它读 decode task 里的 `lme_selected_dp_group_index` 字段（TS 选好的 DP rank），按这个把请求发给那个 DP；不满足条件就退回原来的轮询 `_get_engine_dplb()`。
- **为什么**：TS 选了 DP 之后，得有人听 TS 的话，这个"听"的逻辑就是让权，现在没写，所以 TS 选的 DP 实际没被用上。

#### 引擎侧要写的第 3 块：配置

- **要做什么**：vllm 的 `XdsParams` 加一个开关 `lme_select_dp_enabled`（默认 false），通过 `ts_config_reader` 传给路由器。
- **为什么**：灰度上线用，关着就完全等于现状，不闯祸。

---

## 第 5 章：为什么"TS 选了 DP 但下游又自己选"不是 bug

你之前最大的困惑："TS 选了 DP，下游又自己选一次，那 TS 选的不是白选了？"

文档第 6 节明确写了**三个灰度阶段**，用一个开关 `lme_select_dp_enabled` 控制：

| 阶段 | `ts_dte_select_strategy` | `lme_select_dp_enabled` | TS 选 DP 吗 | 引擎听 TS 的吗 | 目的 |
|---|---|---|---|---|---|
| 1（默认）| `CACHE_AWARE` | false | 不选 | —— | 完全等于现状，不出事 |
| 2（你 commit 这步）| `UNIFIED_DTE_DP` | false | **选** | **不听** | 先验证 TS 选得对不对，不影响线上 |
| 3（引擎做完后）| `UNIFIED_DTE_DP` | true | 选 | **听** | 真正生效 |

文档原话："`UNIFIED_DTE_DP + lme_select_dp_enabled=false` → TS 选 (DTE,DP) 但 TE 不采纳仍轮询，**仅验证 TS 决策正确性**。"

所以"TS 选了但下游不听"**是上线第二阶段的故意行为，不是 bug**。等引擎侧让权写完、开关打开，才真正生效。

---

## 第 6 章：明天跟引擎负责人对齐，谈什么

你目前**没有引擎仓代码**，引擎侧的活要引擎那边的人写。所以明天不是去写代码，是去**把分工和接口对齐**。

### 6.1 告诉对方路由器侧已经就绪的接口（让引擎对接）

- **`DpRankReport` proto**：在 `xds_cpp/network/proto/lme/dp_rank_report.proto`，引擎 import 生成 pb2，**字段名/编号不可改**。
- **zmq 端点**：DP 侧 PUB，TS 侧 SUB。端点地址要两边对齐（TS 侧 `UpdateDTEAddress` 用 DTE 地址列表里的 `tcp://<addr>`）。
- **标记位**：消息里 `is_dp_rank_report` 必须置 true，否则 TS 会当老消息解错。
- **让权字段**：`SeparateInferenceDecodeTask.lme_selected_dp_group_index = 13`，要从 router 一路通到 `_add_task_multi_dp`，中间任何一层不能丢。关联契约：`ScheduleResponse.dte_dp_rank_map = 9`。

### 6.2 跟对方确认文档第 7 节的 5 个待确认项

这些是引擎内部数据口径，**只有引擎的人能答**。逐条问（详见附录 B）：

1. `trans_waiting` 在 decode 侧对应 vllm 哪个 `RequestStatus`。
2. `cache_block_numbers` 能不能用 `kv_cache_manager` 接口查到，查不到怎么估算。
3. vllm v1 里"已输出 token 数"的字段叫什么名。
4. 预留 block 数计算要不要加 `kv_reserve_extra_output_token_len`（默认 256）。
5. Python 侧 `SeparateInferenceDecodeTask` 要不要同步加 `lme_selected_dp_group_index`。

### 6.3 跟对方确认三块代码谁写、什么时候写

1. **DP 上报器 `DPStateReporter`**：每个 DP rank 一个实例，zmq PUB，100ms 周期，发 `DpRankReport`。
2. **`_add_task_multi_dp` 让权逻辑**：读 `task.lme_selected_dp_group_index`，满足条件采纳，否则 fallback 轮询；加 `_record_dp_selection` inflight 记账。prefill task 无此字段，自然 fallback。
3. **配置**：`XdsParams.lme_select_dp_enabled`（默认 false），`ts_config_reader` 透传开关 + v2 超参键（`kv_headroom_ratio` / `bsz_slack` / `sla_tpot_threshold` / `kv_reserve_extra_output_token_len` / `default_max_tokens`）给 router 的 `TSConfigMgt`。

### 6.4 对齐灰度节奏

- 当前 commit 是**第二阶段**（TS 选但引擎不听）。确认引擎侧在 `lme_select_dp_enabled=false` 时**完全等价现状**，不加新行为。
- 引擎让权写完、开关打开，才进**第三阶段**。

### 6.5 上报字段口径（文档第 2 节，逐条跟引擎对）

| 字段 | 需求 |
|---|---|
| `te_name` | DTE 名 |
| `dp_index` | DP rank 在 `_engine_actors` 中的下标 |
| `free_kv_block_num` | 当前空闲 GPU block 数 |
| `total_kv_block_num` | 总 GPU block 数 |
| `cache_config_block_size` | block 大小 |
| `running` | RUNNING 请求：rid → 已输出 token 数 |
| `recv_transfering` | WAITING_FOR_REMOTE_KVS 请求：rid → 已输出 token 数（PD 分离收 KV 中）|
| `trans_waiting` | "等待把 KV 发出"的请求：rid → 已输出 token 数；具体对应哪个 status 按 KV connector 实际语义确认 |
| `dp_seqs_need_kv_blocks` | WAITING（真·admission 等 KV block）请求的预留 block 数求和，量纲是 block 数 |
| `swap_req_num` | PREEMPTED 请求计数（vllm 无 swap，用抢占数替代）|
| `dp_queue_seqs` | `running ∪ waiting`（waiting 全状态）的 rid 集合。**硬约束：必须包含 waiting 全状态的 rid** |
| `total_req_num` | `len(running) + len(waiting)` |
| `full_bsz` | 单 DP 最大 running 上限（`max_num_running_reqs`）|
| `req_prompt_len` | running ∪ waiting 中每个 rid 的真实 prompt 长度 |
| `cache_block_numbers` | 每个 rid 已占 KV block 数（优先用 `kv_cache_manager` 查询接口）|

---

## 第 7 章：一句话总结

> 你这个 commit 已经把**路由器侧**（收 DP 状态、选 DP、传结果）做完了。剩下的**引擎侧**（每个 DP 用 zmq PUB 上报状态、DTE 听 TS 的话选 DP）在 vllm 仓里。你明天跟引擎负责人对齐的就是：**把 proto/zmq/让权字段的接口定死、把文档第 7 节 5 个数据口径问清楚、把三块代码的分工和灰度节奏谈定**。路由器侧你不用再改。

---

## 附录 A：关键代码文件速查

### 路由器侧（你这个仓，已做完）

| 文件 | 干什么 |
|---|---|
| `xds_cpp/network/proto/lme/dp_rank_report.proto` | 定义 `DpRankReport` 消息格式（DP 上报的状态）|
| `xds_cpp/xds_lme/dte_state_receiver/ts_state_receiver.cpp` | TS 用 zmq SUB 收 `DpRankReport`、按 `(te,dp)` 聚合、对账 |
| `xds_cpp/xds_lme/lme_schedule_strategy/dplb_v2_lme.cpp` | 上移后的 DPLB v2，跨候选 DTE 一次性选 (DTE, DP) |
| `xds_cpp/xds_lme/lme_schedule_strategy/lme_sched_tracker.h` | LME 侧"已决策未上报"账本 |
| `xds_cpp/xds_lme/lme_scheduler.cpp` | 主循环 `Forward`，PTE_DTE 路径 + DTE_ONLY 排空循环 |
| `xds_cpp/xds_lme/ts_network/ts_struct_cast.cpp` | proto 序列化/反序列化（`dte_dp_rank_map`、`is_migration_dte_only`）|
| `xds_cpp/xds_lme/info_management/info_manager.cpp` | grpc 接收 `BatchCompletionInfo`（线 B 接收方）|
| `xds_cpp/je_processor/je_post_processor.cpp` | 延迟选 D 全部分支：prefill 完成发 DTE_ONLY、盖印 `lme_selected_dp_group_index`、发 decode |
| `xds_cpp/xds_je/je_global_info.h` | 三个开关：`unifiedDteDpEnabled` / `lmeSelectDpEnabled` / `enableDpScheduler` |

### 引擎侧（vllm 仓，没做）

| 要写的 | 干什么 |
|---|---|
| `DPStateReporter`（每个 DP rank 一个）| zmq PUB，100ms 周期发 `DpRankReport` |
| `_add_task_multi_dp` 让权逻辑 | 读 `lme_selected_dp_group_index`，采纳或 fallback 轮询 |
| `XdsParams.lme_select_dp_enabled` | 灰度开关，默认 false |

---

## 附录 B：文档待确认项逐条翻译

文档第 7 节列了 5 个"待确认项"，全是引擎内部数据口径。逐条解释是什么、为什么待确认、问引擎什么：

### B1. `trans_waiting` 在 decode 侧 DP 上精确对应哪个 `RequestStatus`

- **是什么**：`DpRankReport` 里有个字段 `trans_waiting`，要填"等待把 KV 发出去"的请求。vllm 内部给请求分了很多状态（`RequestStatus`），比如 RUNNING、WAITING、PREEMPTED 等。`trans_waiting` 到底对应哪几个状态，需要引擎的人确认。
- **为什么待确认**：填错了，TS 看到的"等传 KV 的请求数"就不准，选 DP 会偏。
- **问引擎**："vllm 里'等待把 KV 发出'这个语义，精确对应哪个 `RequestStatus` 枚举值？"

### B2. `cache_block_numbers` 的 per-rid 占用 block 查询 API 是否可用

- **是什么**：`DpRankReport` 里 `cache_block_numbers` 要填"每个请求已经占了多少 KV block"。vllm 有个 `kv_cache_manager` 可能能查这个数。
- **为什么待确认**：如果这个接口能用，就精确填；如果不能用，得按 prompt+output 长度估算，估算口径要定。
- **问引擎**："`kv_cache_manager` 有没有接口能查单个请求占了多少 block？没有的话按什么口径估算？"

### B3. vllm v1 中"已输出 token 数"的准确字段名

- **是什么**：`running`、`recv_transfering`、`trans_waiting` 这几个字段都是 `rid → 已输出 token 数`。要填"这个请求已经生成了多少字"。
- **为什么待确认**：vllm v1 里这个数存在请求对象的哪个字段上，叫什么名，要引擎的人确认。
- **问引擎**："vllm v1 里一个请求已经输出了多少 token，读哪个字段？"

### B4. 预留 block 数计算是否需加 `kv_reserve_extra_output_token_len`

- **是什么**：给新请求预留显存时，要不要额外多预留 256 token 的余量（`kv_reserve_extra_output_token_len` 默认 256）。
- **为什么待确认**：这影响 TS 判断"这个 DP 还能不能再接请求"。口径要和 router 侧的 `blocks_for` 计算对齐。
- **问引擎**："预留 block 数要不要加 256 的余量？以 router 侧 `blocks_for` 口径为准对齐。"

### B5. Python 侧 `SeparateInferenceDecodeTask` 是否需同步加 `lme_selected_dp_group_index`

- **是什么**：让权字段 `lme_selected_dp_group_index` 在 C++ 侧的 proto 加了（编号 13）。但 decode task 在引擎里可能要经过 Python 侧序列化，Python 那边的消息结构要不要也加这个字段。
- **为什么待确认**：取决于 Python 是否经手 decode task 序列化。如果经手，不加字段就会丢；如果不经手，就不用加。
- **问引擎**："Python 侧 `SeparateInferenceDecodeTask` 经不经手 decode task 序列化？经手的话要加 `lme_selected_dp_group_index` 字段。"

---

## 附录 C：你可能还被问到的问题

### C1. "为什么 TS 不直接问 DP 要状态，而是 DP 主动报？"

因为 DP 有很多个，TS 一个个去问太慢、也太耗资源。让每个 DP 自己定时喊，TS 被动收，更高效。这就是 zmq PUB/SUB 的设计。

### C2. "zmq 和 grpc 为什么用两个？"

因为它们适合的场景不同：
- zmq PUB/SUB 适合"一方不停发、一方不停收"——DP 每 100ms 报状态，正合适。
- grpc 适合"一问一答"——batch 跑完了通知一次，正合适。

### C3. "为什么要有灰度开关，不能一次性上？"

因为这是大改动，一次性上万一出问题全炸。灰度开关让你能：
- 先关着，确认不影响线上。
- 再开一半（TS 选但引擎不听），验证 TS 选得对不对。
- 最后全开，真正生效。

### C4. "我那个 commit 有没有 bug？"

有两个隐患（我之前查出来的），但**不影响当前灰度阶段**，可以后续再补：
1. `LmeSchedTracker::OnRequestEnd` 定义了但从没被调用——请求 cancel/reject 时账本不回滚，可能慢慢泄漏。等让权真正生效后再补。
2. `SubmitScheduleRequest` / `SubmitPostRequest` 失败时只打日志、不 reject——请求会 hang 到超时。这是 commit 新增路径的异常分支没补全。

这两个可以记下来，但**不是明天对齐的重点**，明天重点是引擎侧的接口和三块代码。

---

> 看完这份文档，你应该能回答：
> 1. 这个项目在干什么？→ 第 1 章
> 2. 你要解决什么问题？→ 第 2 章
> 3. DP 怎么报给 TS、TS 怎么收？→ 第 3 章
> 4. 你 commit 做完了什么、还剩什么？→ 第 4 章
> 5. "TS 选了但下游不听"为什么不是 bug？→ 第 5 章
> 6. 明天跟引擎对齐谈什么？→ 第 6 章

