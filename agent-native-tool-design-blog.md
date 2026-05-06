# 给 AI Agent 造锤子：复杂系统的 Tool 设计框架

> 一台智能巡检设备、一条产线、一个外卖系统——它们向 AI Agent 开放能力时，面临的是同一道题。

---

## 引子：当复杂系统对 AI Agent 开放时

任何复杂系统——智能巡检设备、外卖平台、产线控制系统、金融交易引擎——在设计之初，它的能力都是通过 API 对外暴露的。这些 API 的设计默认了一个前提：**调用者是人。** 人类开发者读文档、理解参数、处理返回值、写 try-catch、调试。人有常识。人会猜。人能查 StackOverflow。

但当调用者从人变成 AI Agent，这套 API 就不够用了。不是 Agent 不够聪明，而是它调用系统的方式和人不一样：

- **人类靠经验填补接口的漏洞。** 文档没写的边界情况，人类会凭常识兜底。Agent 不会——你给它什么，它就信什么。漏一个状态，它就缺一个分支。
- **人类能忍受模糊的返回值。** `{ status: "failed" }` 不足以写程序，但人类会去翻日志、查上下文、试着重试。Agent 拿到这个，很难稳定选对下一步。
- **人类能理解隐式约束。** "这两个接口不能同时调"——人看一眼就懂。Agent 需要它被显式写出来。

这些差异带来的不是"API 好不好用"的问题，而是**系统能不能被 Agent 安全、可靠地编排**的问题。

这不是一个系统独有。智能巡检设备需要 Agent 编排巡检-识别-取证-回到基站。外卖平台需要 Agent 编排下单-支付-分仓-配送。产线需要 Agent 编排排产-投料-加工-质检。金融系统需要 Agent 编排核验-风控-放款。**底层各不相同，但暴露出来的问题是同一套**：状态怎么建模？失败怎么表达？安全边界谁来兜底？调用方需要知道多少底层细节？

这篇文章想回答的问题是：

> **一个复杂系统如何为 AI Agent 重新设计能力接口：Agent 应该看到什么、不该看到什么、每次调用之后能确认什么。**

下面从三种可选路径讲起，再看 Tool 的设计原则、落地方法、常见反模式和验证方式。

---

## 一、为什么 API 直接丢给 Agent 不行

### 1.1 旧方案的典型做法

目前很常见的做法是：**把所有 API 的 JSON Schema 和文档都交给大模型，让它自己理解、自己编排。**

```
tools = [
  { name: "route_upload",    params: { waypoints } },
  { name: "route_start",     params: { route_id } },
  { name: "route_pause",     params: {} },
  { name: "move_to",         params: { lat, lon, alt } },
  { name: "sensor_aim",      params: { pitch, yaw, zoom } },
  { name: "media_capture",   params: { mode } },
  { name: "detect_objects",  params: { image } },
  { name: "speaker_play",    params: { text } },
  // ... 还有几十个类似的
]
system_prompt = "你是一个智能巡检设备控制系统。完成本次巡检任务。"
```

Agent 需要自己把这些原子 API 串成：上传路线 → 启动巡检 → 识别目标 → 暂停路线 → 移动到目标附近 → 调整观测模块 → 拍照取证 → 播报或闪灯 → 恢复路线 → 回到基站。每一步出错还要自己处理恢复路线、释放资源、跳过目标和安全收尾。

这并不是智能巡检设备独有的问题——换成电商系统，API 列表会变成 `inventory_lock`、`payment_charge`、`warehouse_notify`，Agent 同样需要自己串下单-支付-履约-通知。不管是服务端还是物理设备，一旦把实现细节原样喂给调用方，以上交互模式带来的麻烦是相通的。

这个方案有三个互相独立的问题。每一个单独拿出来都可能让系统失败。

---

### 1.2 问题一：步骤数 n 指数级吞噬成功率

每一步都有出错的可能。路线可能上传失败。观测模块可能正被另一个任务占用。目标可能在接近过程中丢失。拍照可能遇到存储空间不足。Agent 每多编排一个步骤，出错的概率就乘一次：

```
总成功率 = P(步骤1) × P(步骤2) × P(步骤3) × ... × P(步骤n)
```

假设每一步成功率 99%（对线上系统来说已经很高了），编排 50 步后：

```
0.99^50 ≈ 0.605   →  总成功率仅 60%
```

如果每一步 95%（链路波动、目标跟踪不稳定、传感器临时不可用时更真实的估计），30 步后直接跌到 21%。

同样的公式放到智能巡检设备场景：每一步 99%，50 步后 60%；每一步 95%，30 步后 21%。**载体完全不同，数学规律完全一样。**

**步骤越多，整体成功率越低。这不是单靠 prompt 优化能解决的，问题出在步骤数本身。**

---

### 1.3 问题二：原始数据侵占上下文，Agent 被迫自己做信息压缩

概率公式只说了"会不会出错"。但还有一个同样致命但更隐蔽的问题：**原子 API 返回的原始数据，会占据大量上下文窗口，而且 Agent 必须自己从中提取有效信号。**

以"判断巡检画面里是否存在需要处理的静止目标"为例。

**原子 API 的返回：**

```json
vision.read_raw_detections(area_id="A-03") →
{
  "frames": [
    { "frame_id": 48291, "ts": 1710484353000,
      "detections": [
        { "class": "vehicle", "bbox": [0.42, 0.31, 0.18, 0.12],
          "confidence": 0.87, "track_id": 3, "position": {"lat": 22.1, "lon": 113.9},
          "velocity_m_s": 0.03 },
        { "class": "person", "bbox": [0.61, 0.26, 0.06, 0.21],
          "confidence": 0.65, "track_id": 9, "position": null,
          "velocity_m_s": null }
      ]
    },
    { "frame_id": 48292, "ts": 1710484353500, "detections": [ ... ] },
    // ... 152 帧
  ]
}
```

Agent 拿到这个需要做什么？
- 遍历 152 帧，过滤 `class == "vehicle"` 且置信度足够的目标
- 按 `track_id` 或空间位置做跨帧去重
- 判断目标是否连续静止超过阈值
- 排除画面边缘、旧缓存、位置不可用的数据
- 标记哪些目标可以驱动后续移动和取证

**这些逻辑全部跑在 Agent 的上下文里。** 152 帧原始数据，每个检测结果的十多个字段——都在消耗 tokens。当 Agent 的上下文被原始数据占满，它用于做真正决策的空间就被挤走了。

**而一个被正确抽象的 Tool：**

```json
target_recognition.list_unprocessed(only_when="stationary_confirmed") →
{
  "count": 2,
  "targets": [
    { "id": "T-003", "type": "vehicle", "stationary_duration_s": 73,
      "position_quality": "rough", "can_drive_movement": true },
    { "id": "T-007", "type": "vehicle", "stationary_duration_s": 91,
      "position_quality": "rough", "can_drive_movement": true }
  ]
}
```

Tool 内部完成了检测→多帧确认→去重→静止判断→质量过滤→返回候选目标。Agent 不需要知道每一帧的 `bbox`、`confidence`、`track_id` 是什么。**Tool 用少量语义化目标，替代了 Agent 上下文里的 152 帧原始数据和处理逻辑。**

同样的压缩发生在任何场景。智能巡检设备：目标检测返回的检测框坐标、置信度、跨帧追踪，全部由 Tool 内部处理完，Agent 只拿到 `has_target: true`。产线：PLC 传感器的时间序列数据由 Tool 内部聚合为"设备健康度：正常/亚健康/故障"。

这就是 Tool 抽象的价值之一：**信息压缩。** 原始系统数据应该先在 Tool 内部变成 Agent 能直接使用的语义信号。否则，Agent 的上下文很快就会被原始数据塞满，真正用于决策的空间反而变少。

---

### 1.4 问题三：隐式约束和状态依赖全部压在 Agent 身上

原子 API 之间不是独立调用的——它们之间有隐式的依赖、时序和互斥关系：

- "先暂停路线，确认暂停成功，才能接管移动控制"——但 `route_pause` 和 `move_to` 是两个独立 API，没有任何东西阻止 Agent 在路线仍运行时发移动指令
- "观测模块未锁定时，拍照取证可能没有目标"——但 `sensor_aim` 和 `media_capture` 是两个独立 API，Agent 可能还没锁定就拍照
- "同一目标处理需要幂等"——但作为两个独立的 `media_capture` 和 `speaker_play` 调用，Agent 可能在网络超时后重试，导致重复取证或重复播报

人类开发者会凭经验感知到这些约束。"暂停路线→移动接近→锁定观测→取证→恢复路线"的顺序、幂等的要求、状态检查——人读过设计文档就懂了。Agent 不会——除非这些约束被显式地写在它看到的接口描述里。而原子 API 的 JSON Schema 不会写这些东西。

同样的约束换到智能巡检设备：观测模块未锁定就拍照没意义，移动中拍照会模糊，电量不足时不该接受移动指令。载体不同，问题类似。

> 典型的原子 API 编排片段（智能巡检设备版）：
>
> ```
> route_pause()                       →  返回 ok
> move_to(target_position)            →  超时，Agent 重试了一次
> move_to(target_position)            →  又返回 ok（设备可能已经偏离原路线）
> sensor_aim(target_id)               →  返回 ok（但目标已经丢失）
> media_capture(mode="photo")         →  返回 ok（拍到的不是目标）
> speaker_play("请立即驶离")           →  返回 ok
> route_resume()                      →  返回 failed（断点已不可恢复）
> // Agent 以为一切正常，实际上：取证错位 + 重复移动 + 路线无法恢复。
> ```
>
> 每一步单看都"成功"了，但整个流程并没有成功。Agent 收到的只是几个 `ok`，中间的目标丢失、重复移动和路线断点失效，它完全不知道。

---

### 1.5 小结：同一个根因的三个面孔

这三个问题——成功率的指数级衰减、原始数据侵占上下文、隐式约束的不可见——看似独立，但指向同一个根因：

> **原子 API 把系统内部的复杂性转嫁给了 Agent。Agent 被迫自己管理状态、压缩信息、推断约束。**

这不是 prompt 工程能解决的问题。这是接口设计层面的问题。

那么——如果旧方案行不通，我们的选项是什么？

---

## 二、三种可选路径

将全 API 系统开放给 Agent，大体上有三条路。每条路在不同的自由度-可靠性张力中取了不同的平衡点。

### 路径 A：全原子 API + 更强的 prompt（继续旧方案）

**做法**：不改造 API，靠优化 prompt、增加 few-shot 示例、扩大上下文来让 Agent 更好地编排。

**优点**：零适配成本。加新 API 就加一条 schema。

**缺点**：第一章的三个问题一个都没解决。步骤数仍然过多，原始数据仍然占满上下文，约束仍然不可见。

**适用**：API 幂等、无状态依赖、容错性高的场景（查数据库、发消息、调 SaaS）。

### 路径 B：预定义工作流 + Agent 只填充参数

**做法**：人预先定义完整的工作流模板。Agent 不参与编排，只在特定判断点介入（如"这张图里的目标是否异常"）。

**优点**：可靠性最高。不符合安全期望的动作不可能被编排出来。

**缺点**：每增加一个新场景 = 写一个新工作流。只能应对"一个场景跑一万次"，不能应对"一万个场景各跑一次"。

**适用**：场景固定、流程稳定、容错要求极低（产线、固定巡检路线）。

### 路径 C：高阶有状态 Tool + Agent 自由编排

**做法**：系统暴露的不再是原子 API，而是高阶、有状态的 Tool。Tool 内部封装了状态机、信息压缩（原始数据→语义信号）、安全门和资源互斥。Agent 在 Tool 提供的能力边界内自由编排。

**优点**：自由度与安全性的折中。Agent 能处理新场景，Tool 保证不出底线错误。上下文不再被原始数据填满。

**缺点**：Tool 设计投入最大。每个 Tool 都必须完整回答 6 个维度的问题（详见第四章）。

**这条路的优势在于适配面更广：常规任务可以用路径 B 的工作流模板跑，全新任务也能用高层 Tool 让 Agent 自己组合出方案。**

```
三条路径对照：

         自由度 ↑
              │
    路径 A    │    路径 C
  (全原子API) │  (高阶有状态Tool)
              │
              │         路径 B
              │     (预定义工作流)
              │
              └──────────────────→ 可靠性
              低                    高
```

### 适用场景迁移

这个框架**不绑定某一种设备载体**。它适用于任何"Agent 需要编排一个带状态、资源、安全约束的复杂系统"的场景：

| 领域 | 典型场景 | Agent 编排什么 |
|---|---|---|
| 智能巡检设备/机器人 | 园区巡检、电力巡检、安防巡检 | 巡检→识别→靠近→取证→处置→回到基站 |
| 微服务系统 | 订单履约、物流调度、退款流程 | 下单→支付→分仓→拣货→配送→签收 |
| 工业自动化 | 产线排程、设备维护、质检流程 | 排产→投料→加工→质检→包装→入库 |
| 金融系统 | 风控审核、批量结算、理赔处理 | 资料核验→风控评分→人工复审→放款 |
| 医疗设备 | 影像采集、报告生成、远程会诊 | 扫描→增强→标注→生成报告→推送医生 |
| 运维/DevOps | 滚动发布、故障自愈、容量伸缩 | 健康检查→摘流→更新→验证→回滚/引流 |

底层都在处理同一组东西：**状态、资源、时序、失败、安全。** 物理载体不同，接口设计维度相同。

---

## 三、Tool 设计的核心原则

### 总原则：职责分界——"这件事该归谁"

Tool 设计首先要回答一个问题：**状态管理、安全校验、信息压缩、业务判断、流程编排，这些事分别该由谁负责？**

答案是按"谁最有能力处理这件事"来分配，而不是按"谁方便"：

| 复杂度类型 | 归谁 | 为什么 |
|---|---|---|
| 资源互斥、操作时序 | **Tool 内部**（Tool 开发者设计） | 确定性逻辑，让不可靠的调用方来管就是灾难 |
| 原始数据→ 有语义的信号 | **Tool 内部** | 不要把 Agent 的上下文浪费在原始数据上 |
| 安全底线（安全区域、电量阈值、资源互斥、幂等） | **系统边界 / Tool 内部**（框架或 Tool 契约强制执行） | 不能依赖任何人"记得检查" |
| 业务判断（是否可疑、优先级、是否人工介入） | **Agent / 上层编排逻辑 / 人** | 随场景变化，Tool 不该理解 |
| 流程编排（先巡检还是先处理告警、先取证还是先播报） | **Agent / 工作流 / 脚本 / 策略引擎** | 跨 Tool 的编排逻辑，Tool 不该知道 |
| 环境状态（系统负载、当前并发数、降级开关） | **系统只读视图** | 服务于编排决策，但不触发动作 |

> **一条总则：确定性逻辑下沉到 Tool，不确定性判断留给 Agent 或上层编排逻辑，安全底线由系统边界或 Tool 契约强制执行。层次放错了，系统要么不可靠，要么不可复用。**

下面四条原则，是这条总原则的展开。

---

### 原则 1：状态优先于动作

**对应总原则中的**："状态机归 Tool 内部，不能归 Agent。"

——

当一条巡检路线的状态是"已完成"，Agent 调了 `pause()`。该不该让它暂停？

答案是：取决于当前状态。如果不定义状态机，Agent 就只能猜——"已完成"的路线调了暂停，到底是拒绝、是重新进入运行状态、还是静默忽略？

动作本身不够，状态才决定动作能不能做。只列方法不定义状态，Agent 就只能猜。

**反例**：接口只写了 `route.pause()` 这个方法名，没说不同状态下调用会怎样。Agent 在"路线切换中"状态下调了 `pause`，结果系统抛了一个异常——Agent 没处理这个分支，整个流程卡死。

**正例**：

```
route.pause() / route.resume() 的状态迁移：

idle        → pause_rejected（"当前没有运行中的路线"）
running     → paused（暂停成功，释放路线控制权）
pausing     → already_pausing（正在暂停，等待稳定状态）
paused      → already_paused（已经暂停，可继续处理目标）
completed   → already_completed（路线已完成，进入收尾）
failed      → resume_rejected（路线失败，需回到基站或人工接管）
```

Tool 的返回值不只是一个状态码，还告诉 Agent "接下来可以做什么"：

```json
// pause() 在 completed 状态下返回：
{ "status": "already_completed", "reason": "route_finished",
  "suggestion": "cleanup_and_return_to_base" }
```

Agent 拿到这个就知道：不能重试暂停，也不该继续处理新目标，而是进入收尾流程。（电商里类似：订单已签收时不能直接取消，而应进入退货或人工处理流程。）

> **原则：所有动作都必须绑定状态迁移。没有状态契约的动作接口是不完整的。**

---

### 原则 2：可预测接口

**对应总原则中的**："Tool 和 Agent 之间的边界必须无意外。"

——

人类开发者遇到模糊的返回值会查日志、看上下文、试着重试。Agent 没有这些手段——它只能根据你给它的接口描述做分支判断。

因此 Tool 的接口必须满足：
- 输入字段稳定，类型严格，单位明确
- 合法范围明确（高度不能越界，距离不能超过安全半径）
- 输出状态可穷举（不是"成功 / 其他"）
- 每个状态有稳定的 data 结构
- 错误不靠隐式 exception 表达
- 降级不伪装成成功

**反例**：

```json
move_to(target_position) →
{ "status": "failed", "message": "Something went wrong" }
```

Agent 不知道是目标位置越界、避障传感器降级、路线控制权未释放，还是链路超时。四种情况的处理方式完全不同，但 Agent 只拿到一个 `failed`，很难选对。

**正例**：

```json
move_to(target_position) →
{
  "status": "rejected",
  "reason": "out_of_safe_zone",
  "recoverable": false,
  "suggestion": "skip_target_and_resume_route",
  "system_state": "route_paused"
}
```

Agent 不需要猜——`recoverable: false` 告诉它不能重试，`suggestion` 告诉它下一步该跳过目标并恢复路线，`system_state: route_paused` 告诉它当前仍停在可恢复状态。（电商里类似：支付失败必须区分余额不足、卡过期、网关超时。）

> **原则：接口的目标不是"容易调用"，而是"没有意外"。**

---

### 原则 3：信息压缩

**对应总原则中的**："原始数据→语义信号归 Tool 内部，不穿过边界。"

——

这在第一章 1.3 节已经展开讲过，这里只强化一条设计准则：

**Tool 的返回值应该是 Agent 可以直接用于决策的语义信号，而不是需要 Agent 二次处理的原始数据。**

反例：`vision.read_raw_detections(...)` 返回 152 帧原始检测数据。

正例：`target_recognition.list_unprocessed(only_when="stationary_confirmed")` 返回少量已验证候选目标。

判断标准很简单：**Agent 拿到返回值之后，是直接可以写 if-else，还是需要先自己做一遍过滤、聚合、去重？** 如果是后者，这个 Tool 就没有完成信息压缩。

> **原则：不要把原始数据直接扔过 Tool 边界。**

---

### 原则 4：安全第一

**对应总原则中的**："安全底线归系统边界或 Tool 契约，不能依赖任何人记得检查。"

——

Agent 有时候会给出超出安全边界的指令。这不一定是 Agent"错了"——可能是它缺少信息，可能是 prompt 没覆盖到这个边界，也可能是模型幻觉。

无论原因是什么，系统的安全边界不能依赖 Agent 自觉。它必须是**在 Tool 框架、Runtime 或 Tool 内部契约中强制执行的、Agent 无法绕过的硬约束**。实现上可以是统一安全门，也可以是具体 Tool 在执行前拒绝；对 Agent 来说，关键是每个不安全请求都得到结构化的拒绝原因，而不是静默执行。

**反例**：目标位置超出安全区域，Agent 不知道有这个限制，直接调了 `move_to`。设备控制服务执行了——因为文档写了"调用方应校验区域"，但 Agent 没读到那行。

**正例**：

```
Agent 调用：move_to(lat, lon, alt)

安全约束自动执行（可以在 Runtime，也可以在 Tool 执行前）：
├── 安全区域检查 → 目标点在安全区域外 → 拒绝，返回 out_of_safe_zone
├── 高度 / 距离检查 → 请求超出设备能力边界 → 拒绝，返回 movement_limit_exceeded
├── 电量检查 → 当前电量不足以完成动作并安全返回 → 拒绝，返回 low_power
├── 资源互斥检查 → 路线控制权尚未释放 → 拒绝，返回 route_still_running
├── 幂等检查 → 同一目标正在处理中 → 拒绝，返回 target_already_processing
└── 全部通过 → 放行给 Tool 执行

# Agent 拿到的拒绝结果是结构化的：
{ "status": "rejected", "reason": "out_of_safe_zone",
  "safe_zone_id": "A-03", "requested": {"lat": 22.1, "lon": 113.9, "alt": 18},
  "suggestion": "skip_target_and_resume_route" }
```

这些检查不能依赖 prompt 里提醒 Agent，也不能依赖调用方"记得先校验"。它们可以被抽成独立安全门，也可以由相关 Tool 内部统一封装，但必须满足同一个对外契约：Agent 无法跳过；冲突时拒绝；拒绝结果可穷举、可处理。

尤其要避免三种假安全：
- **静默修正**：擅自把不合法参数改成合法值但不告诉 Agent。
- **静默忽略**：返回成功但实际上没有执行。
- **静默降级**：用看似成功的状态掩盖实际失败。

> **原则：当 Agent 的指令和安全规则冲突时，以安全规则为准。Agent 可以请求，系统必须校验。不安全请求不能被穿透。**

---

### 四条原则的层次关系

```
总原则：职责分界
    │
    ├── 原则 1：状态优先 ──→ 状态管理归谁
    ├── 原则 2：可预测接口 ──→ 边界契约怎么定
    ├── 原则 3：信息压缩 ──→ 数据怎么过边界
    └── 原则 4：安全第一 ──→ 什么绝对不能过边界
```

四条原则合在一起，回答的是这几个问题：**谁来管状态、边界怎么定、数据怎么传、安全谁兜底。**

---

## 四、从业务到 Tool：设计方法

第三章给了原则。这一章回答"怎么用"——不是拿着框架去套业务，而是从业务出发，反向推导出 Tool。

以下用一个 IoT 设备为例：一台具备移动、观测、识别能力的智能巡检设备，需要通过 Agent 编排完成多种巡检任务。它有物理状态、资源互斥和安全边界，这些问题在很多复杂系统里都会出现。

---

### 4.1 出发点：业务场景，不是 API 列表

不要打开现有 API 文档问"怎么包装"。反过来——**看着业务场景问"Agent 需要什么能力"**。

而且，不是所有 Tool 都必须是高度复合的。一个 Tool 只返回布尔值——`object_detector.has_target(type)` → `true`——如果这个粒度正好匹配 Agent 的决策需要，它就完全合理。**目标不是造最复杂的锤子，是造业务刚好需要的锤子。**

---

### 4.2 列出全部场景的能力需求

把**当前场景 + 可预见的新增场景**全部摊开。每个场景下，Agent 需要"能做什么"——用语义描述，不涉及具体实现。

```
场景 A：标准巡检
  - 按预设路线移动
  - 路线中途暂停和恢复
  - 沿途拍摄影像资料
  - 检测画面中是否存在指定类型的目标
  - 巡检结束后返回出发点

场景 B：定点核查
  - 移动到指定位置
  - 锁定并持续观测某个目标
  - 判断目标是否长时间静止（≥ N 秒）
  - 从多个角度留存影像
  - 核查完成后回到主路线

场景 C：告警响应（未来规划）
  - 接收外部告警事件和位置
  - 中断当前任务，导航到告警位置
  - 到达后扫描周围目标
  - 记录现场画面并回传
  - 处理完毕后恢复原任务
```

这三个场景已经覆盖了"常规任务""子任务干预""紧急打断"三种不同的编排模式。Agent 需要的能力集合必须能支撑全部三类。

---

### 4.3 能力归并：聚合成正交的 Tool

把 4.2 列出的所有能力按**职责**聚类。同一类职责归一个 Tool，不同类拆开。

**首要约束：新增场景时，只新增 Tool 或新增参数，不拆散重组已有的。**

```
原始能力池：
  按路线移动、暂停、恢复、停止、移动到指定点、回到基站、
  拍照、录像开始/停止、检测目标、锁定目标、持续追踪、
  判断静止时长、标记目标已处理、接收告警、中断当前任务、恢复原任务

归并后：

  RoutePlanner        → 按路线执行、暂停、恢复、停止
                        （路线生命周期）
  MoveController      → 移动到指定点、相对目标偏移移动
                        （一次性空间移动）
  DeviceControl       → 启动、回到基站、急停
                        （全局设备控制和最高优先级安全动作）
  MediaStorage        → 拍照、录像开始、录像停止、媒体查询
                        （所有"记录什么"的能力）
  TargetRecognition   → 检测目标、持续追踪、判断静止时长、查询目标、标记已处理
                        （所有"看到什么、目标处于什么生命周期"的能力）
  TargetObservation   → 锁定并观察指定目标
                        （精确观测能力）
  AlertResponder      → 接收告警、生成响应请求、记录告警处理状态
                        （告警响应的事件入口和状态管理）
```

**正交性检验**：
- `RoutePlanner` 和 `MoveController` 有重叠吗？→ 没有。一个管路线生命周期，一个管单次空间移动。
- `MediaStorage` 和 `TargetRecognition` 有重叠吗？→ 没有。一个管记录，一个管识别和目标生命周期。
- 未来加"定点持续监控"场景 → 需要什么？→ 不需要新 Tool，Agent 编排现有的 `MoveController` + `TargetRecognition` + `TargetObservation` + `MediaStorage` 即可。

**反面示范：一个错误的归并**

假设设计者图省事，把"中断当前任务→导航到告警位置→恢复原任务"也塞进 `RoutePlanner`，理由是"都是路线相关的"：

```
RoutePlanner（错误版）：
  按路线执行、暂停、恢复、停止、
  接收告警、中断当前任务、导航到告警位置、恢复原任务
  ← 把告警响应的逻辑也吞了
```

这在当前三个场景下完全能跑通。问题出现在 4.4 的"多设备协同巡检"场景——两台设备分工，需要由一个协调器统一分配任务。告警来了该谁去？如果告警响应逻辑嵌在 `RoutePlanner` 里，协调器无法独立控制"谁去响应告警"——它只能调 `RoutePlanner` 的方法，而 `RoutePlanner` 同时也在管路线。协调逻辑被迫和路线逻辑耦合。

正确的设计是：`AlertResponder` 独立为一个 Tool，协调器可以直接调 `AlertResponder.dispatch(device_id, alert)`。告警入口、路线生命周期和单次移动互不污染。

**归并的铁律**：如果两个能力在未来可能被不同的调用方独立控制，它们就不该在同一个 Tool 里。

**原子 Tool 也是合理的**：
- `TargetRecognition.has_target("vehicle")` → `true`。这就够了。Agent 不需要知道画面里检测框的坐标、置信度、跨帧追踪状态——那些全部封装在 Tool 内部。布尔值是业务需要的准确粒度。

---

### 4.4 扩展性检验

用可预见的未来场景反向拷问。

```
假设新增"多设备协同巡检"场景（两台设备分工覆盖不同区域）：
  - 需要改 RoutePlanner 吗？→ 不需要，每台设备各有一个路线执行器
  - 需要改 TargetRecognition 吗？→ 可能需要新增参数 area_id 来标识检测区域
  - 需要新增 Tool 吗？→ 需要一个 CoordinationManager 来管理任务分配
  - 现有 Tool 需要拆散重组吗？→ 不需要
```

如果答案是"需要拆散 RoutePlanner 和 TargetRecognition 重新组合"——说明当前 Tool 粒度有问题。

**理想状态：新增场景 = 扩展参数 + 少量新 Tool，而不是推翻已有契约。**

---

### 4.5 用第三章原则校验每个 Tool

每个 Tool 逐一过四原则。这是第三章的落地。

以 `RoutePlanner` 为例：

**原则 1（状态优先）— 状态机完整吗？**

```
RoutePlanner 的状态迁移：

idle       ──start(route)──→  running
running    ──pause()──────→  paused
paused     ──resume()─────→  running
running    ──complete─────→  completed
running    ──stop()───────→  stopped
running    ──error────────→  failed  （携带原因：obstacle / link_lost / hardware）
paused     ──stop()───────→  stopped
any        ──emergency────→  aborted  （安全抢占，不可恢复）

每个动作在每种状态下都有明确定义。
例如 paused 状态下 stop() → stopped（释放路线资源）；
completed 状态下 pause() → 返回 already_completed。
```

**原则 2（可预测接口）— 返回值可直接写 if-else 吗？**

```json
route.start(route_id) →
{
  "status": "running",
  "handle": { "route_id": "R-042", "current_waypoint": 3, "remaining": 15 },
  "on_status_change": "register_callback(event)"  // 事件流
}

route.pause() →
{
  "status": "paused",
  "reason": null,
  "resumable": true,
  "current_position": { "x": 12.3, "y": 45.6 }
}
// 或
{ "status": "already_completed", "resumable": false,
  "suggestion": "release_route_and_return" }
```

每个状态的 data 结构稳定，Agent 不需要猜。

**原则 3（信息压缩）— 返回值是语义信号吗？**

`RoutePlanner.status()` 返回 `{ "status": "running", "waypoint": "3/18" }`，而不是把电机转速、实时坐标轨迹、IMU 数据全部抛出来。Agent 需要的是"路线执行到哪里了"，不是"每个轮子转多快"。

**原则 4（安全第一）— 安全门在哪？**

```
Runtime / Tool 安全门：
├── 设备电量 < 移动安全阈值 → 拒绝 start()，返回 low_power_abort
├── 设备当前正在执行另一条路线 → 拒绝 start()，返回 route_already_active
├── 目标位置超出安全边界 → `MoveController.move_to()` 返回 out_of_safe_zone
└── 通信链路中断 → 自动触发安全停止，系统状态 → aborted
```

---

**再看 `TargetRecognition`——一个轻量 Tool 的校验**：

与 `RoutePlanner` 不同，`TargetRecognition` 是轻量级 Tool。这不意味着它可以跳过原则校验——只意味着每个维度的答案更短。

**原则 1（状态优先）— 状态机，即使简单也必须定义：**

```
TargetRecognition 的状态迁移：

stopped ──start(types)────────→ running
running ──on_detect(callback)─→ running  （注册事件回调）
running ──mark_processed(id)──→ running  （更新目标生命周期）
running ──stop()──────────────→ stopped
```

状态不复杂，但每个动作的行为必须明确：`start()` 在 `running` 状态下返回 `already_running`；`mark_processed(id)` 只改变目标生命周期，不偷偷释放其他资源；`list_unprocessed()` 返回未处理目标列表。Agent 不需要靠自己的上下文记住"刚才处理过谁"。

**原则 3（信息压缩）— 这是原子 Tool 最容易翻车的地方：**

假设初版 `TargetRecognition` 的设计者偷了懒：

```json
// 初版（违反原则 3）
recognition.scan() →
{
  "detections": [
    { "class": "vehicle", "bbox": [100,200,300,400], "confidence": 0.87, "track_id": 3 },
    { "class": "vehicle", "bbox": [500,150,700,350], "confidence": 0.92, "track_id": 7 },
    { "class": "person",  "bbox": [800,100,850,400], "confidence": 0.65, "track_id": 2 }
    // ... 更多原始检测结果
  ]
}
```

校验发现：Agent 拿到这个返回值后需要自己过滤 `class`、判断置信度阈值、理解 `track_id` 的跨帧含义。 → **违反原则 3。回退修改。**

修正后：

```json
recognition.on_detect(callback, only_when="stationary_confirmed")
recognition.has_target("vehicle") → true
recognition.list_unprocessed() → { "count": 2, "ids": [3, 7] }
recognition.mark_processed(3, result) → { "status": "ok" }
```

每个返回值都是语义信号。Agent 不需要理解检测框坐标，也不需要自己维护目标去重表。

**原则 4（安全第一）— 原子 Tool 也有安全门：**

```
Runtime / Tool 安全门：
├── 设备不在安全状态时 → 拒绝 scan()，返回 device_not_ready
└── scan() 调用频率 > 上限 → 拒绝，返回 rate_limited，suggestion: wait_and_retry
```

总结：**原子 Tool 的校验清单和复杂 Tool 完全一样，只是每项答案更短。但任何一项答不出——就是设计缺口。**

---

### 4.6 场景覆盖验证

回到 4.2 的三个场景，用设计出的 Tool 把完整流程走一遍。

**场景 A（标准巡检）走查**：

```
Agent 调用 RoutePlanner.start(route_a)
  → 事件通知：waypoint_reached
  → Agent 调 MediaStorage.photo() 拍照留存
  → TargetRecognition.has_target("vehicle") → false，继续
  → 循环直到 route complete
  → DeviceControl.return_to_base()
```
✅ 跑通。

**场景 C（告警响应）走查**：

```
Agent 收到事件：AlertResponder.on_alert(alert)
  → RoutePlanner.pause() 暂停当前巡检
  → MoveController.move_to(alert.location)
  → TargetRecognition.scan() 扫描周围
  → MediaStorage.photo(tags=["alert", alert.id]) 记录现场
  → RoutePlanner.resume() 恢复原巡检
```
✅ 跑通。不需要改任何已有 Tool，编排层新增 AlertResponder 即可。

**场景 B（定点核查）走查——初版跑不通：**

假设设计者在 4.3 归并时漏掉了"一次性移动"和"精确观测"的职责，把路线、移动、观测、拍摄都压进了一个大 Tool。走查场景 B：

```
Agent 调用 MegaPatrol.inspect_target(target_id)
  → 内部暂停路线
  → 内部移动到目标附近
  → 内部锁定目标
  → 内部拍照
  → 内部恢复路线
  → 返回 { "status": "ok", "photos": [...] }
```

❌ 跑是跑通了，但这个 Tool 把路线控制、移动、观测、媒体记录和恢复策略都焊在了一起。未来如果只想换取证角度、改恢复策略、或让告警响应复用"移动到目标"能力，都必须改这个大 Tool。

**回退修正**：回到 4.3，把能力拆回正交 Tool。多角度取证不是 `MediaStorage` 的内部职责，也不是某个固定实现的专属能力；它是上层编排逻辑可以组合出的一个子流程。这个上层可以是 Agent 实时规划、工作流引擎、生成脚本或人写控制逻辑。

再次走查：

```
Agent / 上层编排逻辑：
  → TargetRecognition.on_detect(...) 收到 stationary_confirmed 目标
  → RoutePlanner.pause()
  → MoveController.move_to(target_location)
  → TargetObservation.observe(target_id)
  → MediaStorage.photo(tags=["evidence", target_id, "front"])
  → 上层编排逻辑需要"多角度留证"
  → MoveController.move_to(angle_2) → TargetObservation.observe(target_id) → MediaStorage.photo()
  → MoveController.move_to(angle_3) → TargetObservation.observe(target_id) → MediaStorage.photo()
  → RoutePlanner.resume()
```

✅ 跑通。每个 Tool 的职责仍然单一，复用边界清楚。多角度取证如果经常出现，可以在上层沉淀为一个可复用的 workflow / macro / generated routine，但不应塞进只负责媒体存储的 Tool 里。

---

跑不通 → Tool 有缺口 → 回到 4.2 补能力、4.3 重归并。能闭环，才算设计完成。

本章可以总结为一句话：**先看业务要什么，再把能力归并成正交的 Tool，用原则校验，用场景验证，修缺口，再验证。** 这里的编排可以是实时规划、工作流、生成脚本或其他控制逻辑，不限定某一种实现形态。

---

## 五、反模式：Tool 设计的典型错误

原则告诉人"要做什么"，反模式方便对照自查。下面是 8 个常见错误。

---

### 反模式 1：薄 Wrapper——只是给原子 API 换了个名字

```
# 看起来是"高阶 Tool"，实则什么都没封装
def observe_and_report(target_id):
    device.move_to(target_id.position)
    device.sensor.aim(target_id.position)
    img = device.sensor.capture()
    return device.analyzer.detect(img)
```

**为什么这是反模式**：状态机、资源互斥、操作时序、失败处理——全部甩给了 Agent。代码里没有处理"移动到一半设备没电了怎么办""sensor 正在被另一个任务占用怎么办"。Agent 以为拿到了一个可靠的"观察"能力，实际只是 4 个原子 API 的串联。

**正确做法**：内部封装完整的生命周期——加锁、校验前置条件、处理中层失败、返回结构化结果。Agent 不需要知道内部调了几个 API。

---

### 反模式 2：业务语义焊死在 Tool 里

```
# 错误：把"学校门口目标优先处置"写死在目标识别 Tool 里
def target_recognition.emit(target):
    if target.area_type == "school_zone":
        target.priority = "highest"
        device.speaker.play("请立即驶离")  # 特定业务场景的处置逻辑
    # ...
```

**为什么这是反模式**：明天策略从"学校门口优先"改成"消防通道优先"，就得改 `TargetRecognition`。但这是业务策略，不是目标识别的内在逻辑。换一个巡检场景，优先级规则可能完全不同。

**正确做法**：`TargetRecognition` 只负责提供目标事实和生命周期状态。优先级判断在上层编排逻辑、策略引擎或人工配置里完成，上层拿到结果后决定是否调用 `speaker_play()`、`flash_light()` 或只记录不上报。Tool 保持场景无关，策略保持可替换。（电商里类似：不要把"VIP 优先"写死在订单创建 Tool 里。）

---

### 反模式 3：失败只返回 String

```json
// Agent 拿到这个，什么也做不了
{ "status": "failed", "message": "Something went wrong" }
```

**为什么这是反模式**：同一个 `failed`，可能是安全区域越界、目标暂时丢失、观测模块忙、电量不足。处理方式完全不同，但 Agent 只拿到一个 `failed`，就只能猜。

**正确做法**：每一条错误都要携带三个信息（详见原则 2）：

```json
{ "status": "target_lost", "reason": "out_of_view_after_move",
  "recoverable": false,
  "suggestion": "mark_skip_and_resume_route",
  "system_state": "route_paused" }
```

---

### 反模式 4：隐式副作用——调了 A，悄悄改变了 B

```
# Agent 不知道 mark_processed 居然释放了资源锁
detector.mark_processed(target_id)  # 顺便把观测锁定也释放了
# Agent 还以为 target 在锁着，下一步调 detector.get_status(target_id)
# 返回 target_not_locked——Agent 懵了
```

**为什么这是反模式**：Agent 编排时，脑子里的"系统状态"和真实系统状态产生了漂移。漂移多了，后续判断就会建立在错误假设上。

**正确做法**：每个 Tool 的副作用必须在接口描述里显式声明。如果一个操作会释放资源锁、改变其他 Tool 的可用性、触发异步任务——写清楚。

---

### 反模式 5：Agent 在帮 Tool 记状态

```
# Agent 被迫在上下文中维护"我刚才做了什么"
detector.lock(target_A)             # Agent 记：锁了 A
router.move_to(target_A.position)   # Agent 假设锁住了，但真的锁住了吗？
recorder.capture()                  # 拍的时候可能锁已经丢了
```

**为什么这是反模式**：Agent 变成了状态机的主体。它"以为自己锁定了"，不代表系统真的锁定了。它"以为上一步成功了"，不代表没有静默失败。

**正确做法**：Tool 自己管理状态。Agent 不记"刚才做了什么"——每次做决策前读 Tool 返回的状态。`detector.lock(target_A)` 返回的不只是 `ok`，还包括 `locked_target_id`、`locked_at`、`will_timeout_in_sec`，Agent 可以随时通过 `detector.status()` 获取真实状态，而不是靠自己的记忆。

---

### 反模式 6：调用形态选错

来自一个常见的设计惯性：不管操作是什么性质，全部做成同步调用，靠返回值和异常来传递结果。

**错误 6a：长任务伪装成同步调用**

```
// record_video 要持续录制数分钟，但它被设计成同步调用
result = recorder.record_video(duration_sec=300)
// Agent 在这里阻塞 5 分钟，期间什么都做不了
// 如果中途需要停止录像？没有接口。Agent 只能等它超时或崩溃。
```

**正确**：`record_video` 应该返回一个 handle，Agent 可以中途 `stop()`、查询录制时长、接收"存储空间不足"事件。

**错误 6b：事件流写成轮询**

```
# Agent 每 0.5 秒查一次"有没有新发现"
while True:
    result = detector.scan()
    if result.has_new_target:
        process(result.targets)
    sleep(0.5)
```

轮询浪费上下文、浪费带宽、还会漏事件（两个事件在 0.5 秒间隔内同时发生，后一个覆盖前一个）。

**正确**：`detector.on_detect(callback)`——事件驱动。Agent 注册回调，有新发现时被动通知。

> 判断标准：操作持续占用资源且需要中途控制 → 长任务。事件持续异步发生 → 事件流。其余 → 同步。

---

### 反模式 7：系统环境状态做成查询 Tool

```
# Agent 到处查询环境状态
battery = system.query("battery")
link = system.query("link_quality")
load = system.query("cpu_load")
if battery < 20 or link == "poor":
    return abort()
# ... 50 行之后又查了一遍
```

**为什么这是反模式**：环境状态（电量、链路、系统负载、当前位置）不是"需要 Agent 主动查询的业务数据"——它是 Agent 做每一个编排决策时的背景信息。

做成查询 Tool 有两个问题：一是 Agent 必须在每个决策点手动写查询逻辑，遗漏一个就可能在低电量时继续执行危险操作；二是高频查询浪费上下文和带宽。

**正确做法**：环境状态作为**只读、同步、瞬时的运行时视图**存在，与 Tool 平级但不触发动作。Agent 不需要 `await` 一个查询 Tool，而是在编排决策点直接读取 `system.field`。同时，安全门层或相关 Tool 仍然要在环境状态越过阈值时拒绝危险操作，不能只依赖 Agent 自己检查。

```
# Agent / 上层编排逻辑可以直接读只读视图，不需要 await query Tool：
if system.battery.pct < 20:
    return_to_base()

# 即使上层漏掉检查，移动类 Tool 也必须拒绝危险调用：
{ "status": "rejected", "reason": "low_power", "battery_pct": 18,
  "suggestion": "return_to_base" }
```

---

### 反模式 8：把小模型放在决策位置上

```
# 端侧 4B 模型被要求做业务决策
edge_llm.judge("这个场景是否异常？是否需要上报？是否需要喊话？")
```

**为什么这是反模式**：端侧小模型（4B-9B 参数）的能力边界是感知和分类——"这张图里有没有人""这个设备表面有没有可见裂缝"。它不是规划器，不是决策器，不是业务判断引擎。让它做"是否异常、是否上报、是否喊话"——这是在要求它理解业务上下文、权衡后果、做出价值判断。它做不到，但会给出一个"看起来合理"的答案。

**正确做法**：端侧模型只做一件事——**输入感知数据，输出分类结果**。所有业务决策留给 Agent 的上层编排逻辑、人类策略配置、独立 inference 判断点或结构化规则引擎。

```
端侧模型该做的：  classify(image) → "正常" / "异常" / "不确定"
端侧模型不该做的： decide(image)   → "喊话警告" / "上报指挥中心" / "忽略"
```

---

### 8 个反模式的速查表

| # | 反模式 | 一句话诊断 | 对应原则 |
|---|---|---|---|
| 1 | 薄 Wrapper | Tool 内部没有状态机，Agent 被迫管理一切 | 总原则、原则 1 |
| 2 | 业务语义焊死 | 换场景就要重写 Tool | 总原则 |
| 3 | 失败只返 String | Agent 拿到 `failed` 不知道下一步干嘛 | 原则 2 |
| 4 | 隐式副作用 | Agent 脑子里的系统状态跟实际脱节 | 原则 2 |
| 5 | Agent 帮 Tool 记状态 | Agent 靠记忆而非 Tool 返回值做判断 | 原则 1 |
| 6 | 调用形态选错 | 长任务当同步、事件流当轮询 | 原则 2 |
| 7 | 环境状态成查询 Tool | Agent 到处 await 电量/链路/负载 | 原则 3、原则 4 |
| 8 | 小模型做大决策 | 端侧 4B 被要求判断"是否喊话" | 总原则 |

---

## 六、验证：怎么知道你的 Tool 设计对了

抽象最终要回到真实工作流里验证。以下 3 条可以直接用来检查：

### 1. 端到端闭环测试

用一个完整工作流跑一遍：启动→进入主循环→接收事件→处理中间失败→记录结果→恢复主流程→收尾→上报。跑不通说明 Tool 契约有缺口。

测试时重点检查断裂点：
- 中间失败后能不能恢复？
- 结果能不能上报？
- 中断后能不能接续？

### 2. 场景切换测试

换一个业务场景（比如从"标准巡检"换成"定点核查"），你的 Tool 要不要改？如果需要大改，说明 Tool 里有业务语义泄漏。如果只需要换上层编排逻辑或新增少量 Tool，设计合格。

### 3. 失败路径完备性

随机选一个 Tool、一个状态、一个动作，问："如果这一步失败了，Agent 的下一步是什么？" Agent 能直接从返回结果拿到这个答案，而不是要"猜"——设计合格。

---

## 七、开放问题与未来方向

这套框架回答了很多，但还没回答所有。以下是目前仍在演进中的问题——大部分来自本文撰写过程中的实际争论。

---

### 1. 抽象陷阱与 Escape Hatch

> 一个高层 Tool 封装了 95% 的情况，剩下 5% Agent 需要降级到原子能力手动处理。但这个逃生口的权限和粒度怎么控制？

每个高层 Tool 都可能遇到"设计时没考虑到的边界情况"——这时 Agent 被抽象层锁死了，无法降级处理。一个好的设计应该留一个 escape hatch（逃生口）：当 Tool 返回 `out_of_scope` 时，Agent 可以请求一组受控、审计、限权的降级能力。

这个 escape hatch 的粒度和权限控制还是开放问题——开太大等于回到路径 A（Agent 又可以直接调裸 API），开太小等于抽象陷阱。无论如何，它都不应该暴露底层 SDK 函数名、私有状态机、原始数据流标签或不受状态约束的原子操作。一个可能的方向是限时降级：Agent 获得 N 步受保护操作权限来处理当前异常，完成后自动回到高层 Tool 的编排模式。

---

### 2. Tool 自身能否嵌入智能判断？

> 当前框架假设 Tool 内部只有确定性逻辑，判断留给 Agent。但如果端侧芯片够强，Tool 能不能内嵌一个"判断力"？

比如 `TargetRecognition` 不只返回 `has_target: true`，而是返回 `{ has_target: true, suspicious: true, reason: "静止超过 10 分钟且位于某类业务区域" }`。这个 `suspicious` 是在 Tool 内部完成的——它跨越了"感知"和"判断"的边界。

这会带来两个连锁问题：
- **职责分界是否要重新划？** 前面的总原则说"不确定性判断留给 Agent"。如果 Tool 内嵌判断模型，Agent 负担进一步降低——但 Tool 的可复用性也下降（换一个场景，"可疑"的定义完全不同）。
- **判断的置信度如何传递？** Tool 返回 `suspicious: true`，但这个判断本身有置信度问题。Agent 要不要二次校验？如果无条件信任 Tool 的判断，等于把业务决策权下放到了设备端。如果每个判断都要 Agent 复核，那 Tool 内嵌判断的价值在哪？

当前更保守的边界是：Tool 产出事实和可执行状态，业务判断放在 Agent 的上层编排逻辑、独立 inference 判断点或策略引擎里。Tool 是否能内嵌判断模型，是未来能力变强后才需要重新评估的问题。

---

### 3. 端侧模型足够强时，架构会反转吗？

> 如果端侧能跑 GPT-4 级别的模型，"中心侧编排→端侧执行"这条链路还需要吗？

当前架构的核心假设是：端侧算力有限，只能做感知，不能做规划。但如果这个前提不成立了——设备能不能直接接收高层意图（"去区域 A 看看有没有异常"），在端侧完成 Tool 编排、执行和异常处理？中心侧只负责下发任务目标和接收结果。

好处是离线可用、延迟更低、不依赖链路。风险是端侧模型的可控性和可调试性远不如云端——如果在端侧编排出了一个危险的序列，谁负责叫停？安全门（原则 4）能不能兜住一个在端侧做实时规划的模型？

---

### 4. 长任务是否必要？短任务 + 事件能替代吗？

> 如果系统里同时只有一个实例在运行（单设备、单任务），长任务的 handle 机制是不是过度设计？

对于单设备系统（单巡检设备、单机器人），大多数操作天然是单例的——同一时刻只有一条路线在执行、一个目标在追踪。`route.pause()` 不需要 handle，因为 Agent 知道它在暂停哪个路线。短任务 + 事件流在这些场景下完全够用。

但对于多实例系统（同时跑多个导出任务、多个巡检子任务等待同一组设备资源），handle 提供的同步控制反馈（"这个任务暂停成功了吗？"）无法被事件替代——等事件意味着回调地狱。**什么时候该用哪种，目前更多靠经验判断，缺少系统化的决策框架。**

---

### 5. Multi-Agent 下的 Tool 共享与资源仲裁

> 本文假设的是单 Agent 调用单系统。如果多个 Agent 同时抢同一个资源，谁说了算？

如果多个 Agent 同时调用同一个系统，谁来仲裁资源冲突？Tool 的资源约束在单 Agent 下清晰——声明互斥即可。但在多 Agent 下，互斥本身会引入分布式一致性问题：Agent A 和 Agent B 同时请求锁定同一个资源，谁先拿到？拿不到的 Agent 收到的是"排队等"还是"去问另一个 Agent"？

---

### 6. 编排模型本身的进化

> 确定性工作流用于固定流程没有问题。但如果流程本身需要动态调整——Agent 应该在哪个层面介入编排？

Agent 可以用多种方式编排 Tool：实时规划、预定义工作流、生成脚本、策略引擎，或这些方式的混合。如果业务场景足够多变，确定性工作流可能不如 Agent 实时规划；但固定主流程仍然适合用工作流承载。"中心侧生成控制逻辑" vs "Agent 实时规划"未必是二选一——可能是分层混合：常规流程用预定义工作流跑主流程，分叉点时让 LLM 做子目标规划。但这个分叉点放在哪里、LLM 介入的频率多高——目前没有定论。

---

### 7. Tool 的自动发现与组合

> 人设计 Tool 可以穷举场景。但如果 Agent 自己能发现系统中的能力并组合出新的复合 Tool——这套框架的描述格式够不够？

当前我们讨论的是人工设计 Tool。但如果 Agent 能在运行时发现系统中的能力并自动组合出新的复合 Tool，第四章的归并逻辑就需要从"人类设计师的判断"变成"Agent 可执行的算法"。本文的 Tool 描述格式（状态模型、调用语义、资源约束……）是否足以支撑 Agent 自动判断"这两个能力能归并成一个 Tool 吗"？

---

## 总结

这篇文章的主要观点是：

> **当一个复杂系统要向 AI Agent 开放能力时，不应该把原子 API 文档丢给 AI 让它自己编排，也不应该把所有流程写成死工作流。正确的方式是设计一套高阶、有状态的 Tool，让 Agent 在 Tool 的能力边界内自由编排，而 Tool 或系统边界封装状态机、安全门、信息压缩和资源互斥。**

可以落地成三件事：

1. **1 条总原则 + 4 条设计原则**——职责分界、状态优先、可预测接口、信息压缩、安全第一
2. **从业务到 Tool 的设计方法**——场景能力分解→能力归并成正交 Tool→扩展性检验→原则校验→场景覆盖验证
3. **8 个反模式**——薄 Wrapper、业务语义焊死、失败只返 String、隐式副作用、Agent 记状态、调用形态选错、环境状态成查询 Tool、小模型做大决策

以及一个边界声明：

> 如果场景固定、流程明确（如一条产线、一条固定巡检路线），直接用预定义工作流（路径 B）比自由编排更安全。不是所有场景都需要 Agent 自己编排。选择架构的前提是理解自己场景的确定性与多变性的比例。

---

*本文基于与多个 AI 模型的深度讨论和技术分析整理而成，适用于 IoT 设备控制、微服务编排、工业自动化、金融风控等任何需要 AI Agent 调用复杂系统的场景。*
