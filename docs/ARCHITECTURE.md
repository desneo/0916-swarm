# ARCHITECTURE.md

## 1. 文档目的

本文档定义系统的软件架构、技术选型、模块边界、依赖关系、运行模型、通信方式以及关键架构决策。

本文档回答：

> 系统应该如何组织和实现，以满足 SPEC.md 中定义的系统需求。

本文档不负责定义完整的产品需求、验收条件和业务规则。

### 1.1 文档边界

- `requirement.md`：需求背景、原始材料、讨论过程以及候选方案。
- `SPEC.md`：系统必须提供的功能、行为、约束和验收要求。
- `ARCHITECTURE.md`：系统采用的技术、模块结构、依赖关系、运行模型和架构约束。
- `ADR`：记录重要架构决策及其原因。
- OpenSpec change/design/tasks：针对具体变更的实现设计和任务拆分。

---

# 2. 架构目标

本系统是运行在无人机机载计算平台上的自主任务控制软件。

架构目标：

1. 将任务编排、飞控控制、状态管理、感知、目标管理和通信等职责解耦。
2. 对 PX4、AI 算法、相机、路径规划等外部能力进行适配，而不是将外部系统逻辑耦合进业务模块。
3. 支持 x86 与 RK3588 等不同计算平台。
4. 支持可替换的 AI 算法插件。
5. 支持 GCS 与 UAV 间统一的网络业务通信。
6. 支持任务执行策略独立演进，而不要求修改外部 Task 数据模型。
7. 使用事件驱动机制降低模块之间的耦合。
8. 保持单进程架构，避免在当前系统规模下引入不必要的进程间通信复杂度。
9. 保持模块高内聚、低耦合，并明确依赖方向。
10. 允许后续根据性能验证结果调整线程和执行模型，而不提前固定线程数量。

---

# 3. 架构原则

## 3.1 高内聚、低耦合

模块应围绕明确的职责组织。

模块之间通过稳定的能力接口或事件进行协作，避免直接依赖其他模块的内部实现。

---

## 3.2 同步能力调用与异步事件传播分离

系统内部采用两种主要通信方式：

### 同步 C++ 调用

用于：

- 查询当前状态；
- 获取当前目标信息；
- 调用明确的能力；
- 需要立即得到结果的操作。

例如：

```text
Task Execution Engine
        │
        ├── 查询 UAV State
        ├── 查询 Target
        └── 调用 Flight Control Capability
```

### eventcpp 异步事件

用于：

- 状态变化；
- 执行结果；
- 异步控制请求；
- 模块间事件通知；
- 不要求调用方立即获得结果的消息传播。

---

## 3.3 网络通信与进程内通信分离

网络通信统一使用 Zenoh。

进程内部不使用 Zenoh 作为模块间通信机制。

```text
进程外 / 网络
        │
      Zenoh
        │
        ▼
   Communication
        │
        ▼
    eventcpp / C++
        │
        ▼
   Business Modules
```

这样可以避免把网络通信机制泄漏到业务模块内部。

---

## 3.4 业务逻辑与外部技术解耦

业务模块不直接依赖：

- Zenoh API；
- MAVSDK 具体实现；
- PX4/MAVLink 细节；
- Pluma 加载细节；
- AI 算法具体实现；
- GeographicLib API；
- ZLMediaKit API。

这些能力通过内部抽象或 Adapter 进行隔离。

---

## 3.5 Task 与 Command 分离

Task 表达：

> 系统需要完成什么任务。

Command 表达：

> 向飞控发送什么直接控制能力。

Task 不包含 Command 列表。

Task 的具体执行策略由机载系统根据 Task 类型决定，而不是由 GCS 指定 BehaviorTree 结构。

---

## 3.6 Task Scheduler 与 Task Execution Engine 分离

Task Scheduler 负责：

- 当前 Task 管理；
- Task 生命周期；
- Task 替换；
- 启动执行；
- 中止执行；
- 接收执行结果。

Task Execution Engine 负责：

- 根据 Task 类型确定执行策略；
- 创建/运行 BehaviorTree；
- 管理 Node；
- 执行实际任务；
- 返回执行结果。

Scheduler 不负责具体 BehaviorTree。

---

# 4. 系统边界

系统内部负责：

- Task 管理与执行；
- Command 调度；
- UAV 状态管理；
- 安全状态监控；
- 飞控能力调用；
- 图像获取与 AI 算法编排；
- Target 管理；
- 视频输出；
- GCS/UAV 网络通信；
- 插件生命周期管理。

系统外部提供：

- PX4 飞控；
- MAVLink；
- MAVSDK 所连接的飞控；
- MIPI Camera；
- AI 算法插件实现；
- GCS；
- 其他 UAV；
- ZLMediaKit；
- 外部路径规划等能力。

---

# 5. 技术选型

| 领域 | 技术 | 主要职责 |
|---|---|---|
| 编程语言 | C++17 | 系统主要实现语言 |
| 构建 | CMake | 构建与交叉编译 |
| 网络业务通信 | Zenoh / zenoh-cpp | GCS/UAV 网络通信 |
| 进程内事件 | eventcpp | 模块间异步事件传播 |
| 飞控 | PX4 | 飞行控制器 |
| 飞控接口 | MAVSDK | 机载系统与 PX4 的能力接口 |
| 飞控协议 | MAVLink | 飞控通信协议 |
| 任务执行 | BehaviorTree.CPP | Task 执行策略与行为编排 |
| 图像处理 | OpenCV | 图像与 `cv::Mat` 基础能力 |
| 地理计算 | GeographicLib | WGS84 等地理计算 |
| 插件系统 | Pluma | 可替换插件加载与管理 |
| 日志 | spdlog | 系统统一日志 |
| 视频服务 | ZLMediaKit | 视频流服务与 RTSP 输出 |

技术选型遵循：

> 稳定的第三方能力通过内部抽象隔离，业务代码不直接绑定第三方库 API。

---

# 6. 总体架构

系统采用四层结构：

```text
┌──────────────────────────────────────────────┐
│                 Core Runtime                 │
│      生命周期 / 配置 / 日志 / Event Bus      │
└──────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────┐
│              Capability Modules             │
│                                              │
│ Task Scheduler                               │
│ Task Execution Engine                        │
│ UAV State Management                         │
│ StateMonitor                                 │
│ Target Management                            │
│ Flight Control                               │
│ Perception                                   │
│ Video Streaming                              │
│ Communication                               │
└──────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────┐
│              External Adapters               │
│                                              │
│ MAVSDK / MAVLink / PX4                       │
│ Zenoh                                        │
│ MIPI Camera                                  │
│ ZLMediaKit                                   │
│ GeographicLib                                │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│                  Plugins                     │
│                                              │
│ Detection Plugin                             │
│ Tracking Plugin                              │
│ 其他可替换算法/能力插件                      │
└──────────────────────────────────────────────┘
```

Plugin 层是可替换实现层，不属于 Capability Modules 的下级业务模块。

---

# 7. Core Runtime

Core Runtime 是系统级运行框架，不承担具体业务逻辑。

主要职责：

- 系统启动；
- 系统关闭；
- 模块生命周期协调；
- eventcpp 初始化；
- Plugin Management 初始化；
- 日志初始化；
- 基础配置加载；
- 系统运行状态管理；
- 模块启动顺序管理。

Core Runtime 不应成为业务逻辑集中处理的“上帝模块”。

每个模块负责自己的初始化、运行和释放逻辑，Core Runtime 只负责生命周期协调。

---

# 8. Task Scheduler

Task Scheduler 负责 Task 生命周期管理。

主要职责：

- 接收 Task；
- 管理当前 Task；
- 管理 Task 状态；
- 启动 Task Execution Engine；
- 中止当前 Task；
- 处理新 Task 对当前 Task 的替换；
- 接收 Task Execution Engine 返回的执行结果；
- 对外提供当前 Task 状态。

Task Scheduler 不负责：

- 决定 BehaviorTree 结构；
- 选择具体 Node；
- 直接控制飞控；
- 执行 AI 算法；
- 判断具体飞行行为。

Task Scheduler 与 Task Execution Engine 之间通过异步事件传递执行结果。

---

# 9. Task Execution Engine

Task Execution Engine 是 Task 的实际执行层。

主要职责：

1. 根据 Task 类型选择执行策略；
2. 创建或获取对应 BehaviorTree；
3. 执行 BehaviorTree；
4. 管理 Node；
5. 调用其他能力模块；
6. 根据实际系统状态判断 Node 执行结果；
7. 向 Task Scheduler 异步返回执行结果。

Task Execution Engine 可以直接调用：

- Flight Control；
- UAV State Management；
- Perception；
- Target Management。

这些调用属于明确的同步能力调用，不要求全部转换成 eventcpp。

---

## 9.1 Task 与 BehaviorTree

Task 只表达任务意图和必要参数。

Task 不包含 BehaviorTree 结构。

例如：

```text
GCS
 │
 │ Task
 ▼
Task Scheduler
 │
 ▼
Task Execution Engine
 │
 ├── Task Definition / Strategy Mapping
 │
 └── BehaviorTree
       │
       ├── Node
       ├── Node
       └── Node
```

Task 类型到具体执行策略的映射属于机载系统内部实现。

因此：

> GCS 不需要知道具体 BehaviorTree 结构。

同一个 Task 类型可以在不修改外部 Task 模型的情况下替换内部执行策略。

---

# 10. Command

Command 是直接作用于 Flight Control 的低层控制输入。

Command 不属于 Task 数据结构。

Command 不需要独立的 Task 生命周期管理，也不需要持续向 GCS 报告 Command 执行状态。

Command 的基本路径：

```text
GCS / Task Execution
        │
        ▼
    Flight Control
        │
        ▼
      PX4
```

如果 Command 无法成功发送给 Flight Control，应产生错误/告警事件，使 GCS 能够感知。

需要特别区分：

```text
Command Dispatch Success
        ≠
Actual Flight Action Completed
```

如果 Command 由 BehaviorTree Node 使用，则 Node 负责继续观察 UAV 实际状态，并根据实际状态决定：

```text
RUNNING
SUCCESS
FAILURE
```

---

# 11. UAV State Management

UAV State Management 是系统统一的 UAV 状态来源。

维护：

- 本机 UAV 最新状态；
- 其他 UAV 最新状态；
- GCS 相关状态。

状态来源包括：

- Flight Control；
- UAV-to-UAV Communication；
- GCS Communication。

不同来源的数据统一进入 State Management。

对于实时状态：

> 最新有效状态具有权威性。

主要能力：

- 状态接收；
- 状态更新；
- 最新状态保存；
- 同步状态查询；
- 状态变化事件发布。

State Management 不负责：

- 安全决策；
- Geofence 判断；
- Task 成功/失败判断；
- Target 管理。

---

# 12. StateMonitor

StateMonitor 负责系统运行状态和安全相关状态监控。

主要职责：

- UAV 状态监控；
- 系统状态监控；
- Geofence 检查；
- 安全状态变化检测；
- 产生安全相关事件。

Geofence 属于 StateMonitor 的职责范围。

StateMonitor 不负责：

- Target 状态监控；
- Target 有效性判断；
- AI 业务逻辑；
- Task 编排；
- 直接控制 PX4。

StateMonitor 发现安全状态变化后，通过 eventcpp 或定义好的能力接口通知相关执行/安全处理模块。

---

# 13. Flight Control

Flight Control 是机载业务系统与飞控之间的能力抽象。

内部链路：

```text
Flight Control
      │
      ▼
    MAVSDK
      │
      ▼
   MAVLink
      │
      ▼
     PX4
```

支持的飞控能力包括：

- Arm；
- Disarm；
- Takeoff；
- Land；
- RTL；
- Waypoint；
- Route；
- Hover；
- Stop Motor；
- NED Velocity。

Flight Control 负责：

- MAVSDK 集成；
- MAVLink/PX4 交互；
- 飞控连接生命周期；
- 飞控状态获取；
- 飞控错误转换；
- 飞控能力抽象。

Flight Control 不负责：

- Task 编排；
- BehaviorTree；
- Task 成功/失败判断；
- 高层业务策略。

---

# 14. Perception

Perception 负责机载图像输入以及 AI 算法能力编排。

总体结构：

```text
MIPI Camera
     │
     ▼
 Perception
     │
     ├──────────────┐
     ▼              ▼
Detection        Tracking
 Plugin           Plugin
     │              │
     └──────┬───────┘
            ▼
       Perception
         Results
```

## 14.1 图像输入

图像直接从 MIPI Camera 获取。

Perception 每次获取最新的 `cv::Mat`。

系统不建立图像历史缓存队列。

---

## 14.2 AI Plugin

AI 算法插件分为：

- Target Detection；
- Single Target Tracking。

Detection 与 Tracking 是两个独立能力，不强制规定固定流水线关系。

AI Plugin 负责算法本身。

Perception 负责：

- 相机生命周期；
- 最新图像获取；
- AI Plugin 调度；
- Detection/Tracking 控制；
- 结果事件发布。

---

## 14.3 Tracking

Perception 对外提供业务级 Tracking 控制能力。

其他模块可以请求：

- 启动指定目标 Tracking；
- 停止 Tracking；
- 根据任务需要改变 Tracking 状态。

内部控制请求优先通过 eventcpp 进行异步传播。

Detection/Tracking 结果也通过 eventcpp 向相关模块发布。

---

# 15. Target Management

Target Management 是系统统一的 Target 信息来源。

维护两类 Target：

### 15.1 机载感知 Target

保存当前最新信息。

不保存历史轨迹。

### 15.2 GCS Target

保存：

- 当前 Target 信息；
- 每个 Target 最近 20 条历史记录。

Target Management 不负责：

- Detection；
- Tracking；
- AI 算法；
- UAV State；
- Geofence。

需要 Target 信息的其他模块应从 Target Management 查询，而不是直接访问 Detection/Tracking Plugin。

轨迹评价属于独立能力，不由 Target Management 承担。

---

# 16. Video Streaming

Video Streaming 独立于 Perception。

基本数据流：

```text
Perception
    │
    │ cv::Mat + Target Overlay Information
    ▼
Video Streaming
    │
    ├── Encoding
    ├── Packaging
    └── ZLMediaKit Adapter
            │
            ▼
          RTSP
```

Perception 不直接依赖 ZLMediaKit。

Video Streaming 负责：

- 接收需要输出的视频图像；
- 视频编码；
- 视频封装/传输；
- ZLMediaKit 集成；
- RTSP 输出。

具体 Codec、MPP、buffer、zero-copy、RTP packetization 等实现细节不在当前架构层固定，待实现阶段根据平台性能需求确定。

---

# 17. Communication

Communication 是系统统一网络通信基础设施。

网络通信技术：

> Zenoh / zenoh-cpp

主要职责：

- Zenoh Session 生命周期；
- 网络连接；
- 消息发送；
- 消息接收；
- ACK；
- 超时；
- 重试；
- 网络状态；
- 外部消息与内部消息之间的适配。

Communication 不负责：

- Task 业务逻辑；
- Target 业务逻辑；
- AI 业务逻辑；
- Flight Control 业务逻辑。

---

## 17.1 GCS 通信

主要承载：

- Task；
- Command；
- 本机 UAV State；
- 当前 Task State；
- Alert。

---

## 17.2 UAV-to-UAV 通信

主要承载：

- 本机 UAV State；
- 其他业务信息。

最多支持约 8 架 UAV 的业务通信规模。

通信策略：

- 优先单播；
- 尽量减少广播；
- 实时状态允许丢失；
- 最新有效状态具有权威性。

---

## 17.3 可靠消息

Task、Command 和控制类消息需要可靠传输机制以及 ACK。

ACK 表示：

> 消息已经被接收。

不表示：

- Task 已开始；
- Task 执行成功；
- Task 已完成。

---

# 18. Plugin Management

Plugin Management 使用 Pluma。

Plugin Manager 负责：

- Plugin 发现；
- Plugin Library 加载；
- Plugin 生命周期管理；
- Plugin 对象创建；
- Plugin 释放。

插件库加载与 Plugin Object 实例化分离。

可以在系统启动阶段加载插件库，但 Plugin Object 默认采用按需实例化方式。

---

## 18.1 Plugin 依赖原则

业务模块依赖：

> Plugin Interface

而不是：

> Plugin Implementation

Plugin Manager 负责实现的加载和生命周期。

业务模块不应将 Plugin Manager 作为全局 Service Locator 使用。

---

# 19. BehaviorTree 与 Plugin 的边界

BehaviorTree.CPP 与 Pluma 解决的是不同问题。

### BehaviorTree.CPP

解决：

> 如何组织和执行 Task。

### Pluma

解决：

> 如何替换具体算法或能力实现。

因此：

```text
Task Execution Engine
        │
        ▼
 BehaviorTree.CPP
        │
        ▼
      Node
        │
        ├── Flight Control
        ├── Perception
        ├── Target Management
        └── UAV State
```

而：

```text
Perception
    │
    ▼
 Plugin Interface
    │
    ▼
   Pluma
    │
    ├── Detection Plugin
    └── Tracking Plugin
```

BehaviorTree 本身不是 Pluma Plugin。

---

# 20. Geographic Computation

系统使用 GeographicLib 提供地理计算能力。

为了避免业务模块直接依赖 GeographicLib：

```text
Business Module
      │
      ▼
Geographic Tool
      │
      ▼
GeographicLib
```

Geographic Tool 作为系统内部统一地理计算能力。

业务模块不得直接调用 GeographicLib API。

---

# 21. 模块通信模型

系统采用以下原则：

| 场景 | 机制 |
|---|---|
| 当前状态查询 | C++ 同步调用 |
| 当前 Target 查询 | C++ 同步调用 |
| 调用明确能力 | C++ 同步调用 |
| 状态变化 | eventcpp |
| 异步执行结果 | eventcpp |
| 异步控制请求 | eventcpp |
| GCS ↔ UAV | Zenoh |
| UAV ↔ UAV | Zenoh |

因此形成三层通信边界：

```text
                    Network
                       │
                     Zenoh
                       │
              Communication Module
                       │
                  eventcpp
                       │
            ┌──────────┴──────────┐
            │                     │
       Async Events        Sync C++ Calls
            │                     │
            └──────────┬──────────┘
                       │
                 Business Modules
```

---

# 22. 主要依赖方向

推荐依赖方向：

```text
Core Runtime
      │
      ▼
Business / Capability Modules
      │
      ├── Internal Capability Interfaces
      │
      ├── Plugin Interfaces
      │
      └── External Adapters
               │
               ▼
        External Technologies
```

必须避免：

```text
External Adapter
      │
      ▼
Business Module
```

以及：

```text
Business A
   │
   ▼
Business B
   │
   ▼
Business A
```

形成循环依赖。

Task Execution Engine 可以调用 Flight Control、UAV State Management、Perception、Target Management。

被调用模块不得反向直接依赖 Task Execution Engine。

如果需要异步返回结果，应使用 eventcpp。

---

# 23. 并发模型

系统采用：

> 单进程 + 少量长期运行线程 + eventcpp 事件驱动。

不采用：

> 一个模块一个线程。

---

## 23.1 BehaviorTree

BehaviorTree 使用统一的执行/tick 模型。

不为每个 BehaviorTree 创建独立线程。

---

## 23.2 独立线程场景

以下任务可以根据实际性能和阻塞特性使用独立执行线程：

- 阻塞式 IO；
- AI 推理；
- 图像处理；
- 视频编码；
- 其他不可控耗时操作。

线程数量不在当前架构阶段固定。

最终线程模型根据：

- CPU 利用率；
- 调度延迟；
- AI 推理耗时；
- 视频处理耗时；
- 网络阻塞；
- 实际系统性能测试

确定。

---

## 23.3 Event Thread 约束

核心事件线程不得执行：

- 不可控的长时间阻塞操作；
- 大量计算；
- 长时间 AI 推理；
- 长时间视频编码。

---

# 24. 生命周期

系统启动大致遵循：

```text
Process Start
     │
     ▼
Logging Init
     │
     ▼
Configuration Init
     │
     ▼
eventcpp Init
     │
     ▼
Plugin Manager Init
     │
     ▼
External Adapter Init
     │
     ▼
Capability Module Init
     │
     ▼
Task / Runtime Services Start
     │
     ▼
System Ready
```

关闭过程按照反向依赖关系执行。

模块自身负责：

- init；
- start；
- stop；
- release。

Core Runtime 负责协调顺序。

---

# 25. 异常与恢复

系统异常处理遵循：

> 外部异常 → Adapter 转换 → Capability Module → Business/Event → 上层处理。

例如：

```text
PX4 / MAVSDK Error
        │
        ▼
Flight Control
        │
        ▼
统一错误信息
        │
        ▼
eventcpp
        │
        ▼
Task / StateMonitor / Communication
```

业务模块不直接处理第三方库异常细节。

---

# 26. 部署模型

当前采用单进程部署：

```text
┌────────────────────────────────────────────┐
│              UAV Application              │
│                                            │
│ Core Runtime                               │
│ Task Scheduler                             │
│ Task Execution Engine                      │
│ UAV State Management                       │
│ StateMonitor                               │
│ Target Management                          │
│ Flight Control                             │
│ Perception                                 │
│ Video Streaming                            │
│ Communication                              │
│ Plugin Manager                             │
│                                            │
│ Dynamic Plugin Libraries                   │
└────────────────────────────────────────────┘
```

外部系统：

```text
        PX4
         │
      MAVLink
         │
     Application
         │
   ┌─────┴─────┐
   │           │
  GCS        UAVs
```

---

# 27. 工程约束

## 27.1 编程语言

使用：

> C++17

---

## 27.2 构建

使用：

> CMake

支持：

- x86；
- RK3588；
- 交叉编译。

第三方依赖优先通过 CMake `find_package` 管理。

当依赖需要向下游传播时，应正确使用 `PUBLIC` / `PRIVATE` / `INTERFACE`。

---

## 27.3 日志

统一使用：

> spdlog

禁止：

```cpp
std::cout
printf
```

项目代码应通过统一日志封装输出。

日志应包含定位问题所需的上下文信息。

日志文件：

- 单文件最大约 50 MB；
- 保留 5 个历史文件。

程序启动时输出构建版本等 Build Information。

---

## 27.4 命名空间

项目统一使用：

```cpp
namespace plane
```

---

## 27.5 格式化

使用 clang-format。

当前代码风格基础：

- Google Style；
- 缩进 4；
- Column Limit 120；
- Pointer Alignment：Left。

---

# 28. 架构级禁止事项

以下行为必须经过架构变更后才能进行：

1. 将单进程改为多进程。
2. 改变 Zenoh 作为网络业务通信基础设施的定位。
3. 将 eventcpp 替换为其他内部通信机制。
4. 改变 PX4 + MAVSDK + MAVLink 的飞控集成主路径。
5. 将 BehaviorTree.CPP 从 Task Execution Engine 中移除或替换。
6. 将 Task Scheduler 与 Task Execution Engine 合并。
7. 将 Task 改造为包含 Command 序列。
8. 让 GCS 直接定义机载 BehaviorTree。
9. 让 Perception 直接依赖 ZLMediaKit。
10. 让业务模块直接依赖具体 AI Plugin 实现。
11. 让业务模块直接依赖 GeographicLib。
12. 引入模块间循环依赖。
13. 将一个模块固定绑定一个独立线程作为通用架构原则。

发生上述变化时，应先修改本文档，并通过 ADR 记录原因。

---

# 29. 架构决策记录 ADR

ADR 用于记录已经确认的重要架构决策。

每个 ADR 至少包含：

- Context：背景；
- Decision：决策；
- Alternatives：候选方案；
- Rationale：选择原因；
- Consequences：影响。

---

## ADR-001：采用单进程架构

### Context

系统包含多个业务模块和插件，需要进行高频状态交互、图像处理以及任务执行。

### Decision

采用单进程架构。

### Alternatives

- 多进程；
- 微服务化；
- 单进程。

### Rationale

当前系统规模下，模块之间存在大量高频状态和能力调用。

单进程可以：

- 降低 IPC 复杂度；
- 简化生命周期管理；
- 简化状态共享；
- 降低通信开销；
- 方便插件动态加载。

网络边界仍通过 Zenoh 与外部系统通信。

### Consequences

优点：

- 结构简单；
- 低通信开销；
- 易于共享状态；
- Plugin 可以直接加载。

代价：

- 单个模块崩溃可能影响整个进程；
- 模块隔离程度低于多进程架构。

---

## ADR-002：网络通信采用 Zenoh

### Context

系统需要同时支持：

- GCS ↔ UAV；
- UAV ↔ UAV；
- 可靠控制消息；
- 实时状态消息。

### Decision

采用 Zenoh / zenoh-cpp 作为统一网络业务通信基础设施。

### Alternatives

- ROS 2 DDS；
- MQTT；
- 自定义 UDP；
- TCP 自定义协议；
- Zenoh。

### Rationale

Zenoh 可以同时覆盖：

- 网络通信；
- Pub/Sub；
- 请求/响应；
- 数据分发；
- 不同网络环境下的通信。

同时避免让业务模块直接依赖具体传输协议。

### Consequences

Communication 模块承担 Zenoh 适配职责。

业务模块不直接使用 Zenoh API。

---

## ADR-003：进程内采用 eventcpp

### Context

系统内部需要大量状态变化、执行结果和异步控制事件。

### Decision

使用 eventcpp 作为进程内异步事件总线。

### Alternatives

- Zenoh；
- ROS 2；
- 自定义 Observer；
- eventcpp。

### Rationale

系统采用单进程架构，没有必要使用网络通信机制解决纯进程内事件传播问题。

eventcpp 可以降低模块之间的直接依赖。

### Consequences

形成明确边界：

```text
Network      → Zenoh
Process      → eventcpp
Sync Call    → C++
```

---

## ADR-004：飞控采用 PX4 + MAVSDK + MAVLink

### Context

系统需要访问飞控能力并获得 UAV 实时状态。

### Decision

采用：

```text
Business
   ↓
Flight Control
   ↓
MAVSDK
   ↓
MAVLink
   ↓
PX4
```

### Alternatives

- 直接 MAVLink；
- MAVSDK；
- ROS 2 飞控接口；
- PX4 原生接口。

### Rationale

MAVSDK 为上层提供更稳定的能力抽象，同时 MAVLink 保持与 PX4 的标准通信路径。

### Consequences

Flight Control 屏蔽 MAVSDK/MAVLink/PX4 细节。

---

## ADR-005：采用 BehaviorTree.CPP 作为 Task 执行框架

### Context

Task 需要根据任务类型执行复杂的机载行为，并根据实际 UAV 状态判断行为完成情况。

### Decision

使用 BehaviorTree.CPP。

### Rationale

BehaviorTree 可以将：

- 条件；
- 动作；
- 状态；
- 成功；
- 失败；
- RUNNING

组织成明确的执行策略。

同时可以让 Task 外部模型与内部执行策略解耦。

### Consequences

Task 不包含 BehaviorTree。

Task Execution Engine 负责 Task → Execution Strategy → BehaviorTree。

---

## ADR-006：Task Scheduler 与 Task Execution Engine 分离

### Context

Task 生命周期管理和实际 Task 执行属于不同职责。

### Decision

拆分：

```text
Task Scheduler
       │
       ▼
Task Execution Engine
```

### Rationale

Scheduler 负责“管理什么 Task 正在运行”。

Execution Engine 负责“这个 Task 具体怎么执行”。

两者分离后，可以在不改变 Task 生命周期管理逻辑的情况下替换执行策略。

### Consequences

执行结果通过 eventcpp 返回 Scheduler。

---

## ADR-007：Task 不包含 Command

### Context

Task 表达高层任务意图，而 Command 是低层飞控控制输入。

### Decision

Task 与 Command 分离。

### Rationale

如果 Task 包含 Command，则 GCS 需要了解机载内部执行策略，导致：

- GCS 与机载实现耦合；
- Task 模型不稳定；
- 内部策略难以演进。

因此 Task 只表达任务意图。

### Consequences

具体行为由机载 Task Execution Engine 决定。

---

## ADR-008：采用 Pluma 作为 Plugin Framework

### Context

AI Detection、Tracking 等算法需要支持替换。

### Decision

采用 Pluma。

### Rationale

插件实现应与业务模块解耦。

核心系统依赖稳定的 Plugin Interface，而不是具体算法实现。

### Consequences

Plugin Manager 统一负责插件库加载和对象生命周期。

---

## ADR-009：Perception 使用最新帧模型

### Context

机载 AI 感知更关注当前图像，而不是完整视频历史。

### Decision

MIPI 图像输入采用：

> 持续采集 + 获取最新帧

不建立图像历史缓存队列。

### Rationale

避免旧帧积压导致：

- 感知延迟；
- 内存增长；
- 算法处理速度与采集速度不匹配时的延迟累积。

### Consequences

AI 模块获取的是当前最新有效图像。

---

## ADR-010：Detection 与 Tracking 作为独立 Plugin

### Context

不同任务可能需要 Detection 或 Tracking，也可能在不同阶段使用不同算法。

### Decision

Detection 和 Tracking 分别作为独立 Plugin 能力。

### Rationale

避免固定：

```text
Detection → Tracking
```

流水线。

Perception 根据业务需求控制两种能力。

### Consequences

未来可以独立替换 Detection 或 Tracking 算法。

---

## ADR-011：Perception 与 Video Streaming 解耦

### Context

Perception 负责 AI 感知，而视频输出属于独立媒体能力。

### Decision

Video Streaming 独立为 Capability Module。

### Rationale

避免 Perception 直接依赖：

- 编码器；
- ZLMediaKit；
- RTSP。

这样可以独立演进 AI 和视频链路。

### Consequences

数据流：

```text
Perception
    ↓
Video Streaming
    ↓
ZLMediaKit
    ↓
RTSP
```

---

## ADR-012：统一 Geographic Tool

### Context

多个模块可能需要 WGS84 地理计算。

### Decision

使用 GeographicLib，并通过统一内部 Geographic Tool 暴露。

### Rationale

避免业务模块直接依赖第三方地理计算库。

### Consequences

未来替换 GeographicLib 时，不需要修改业务模块。

---

## ADR-013：同步能力调用与异步事件分离

### Context

系统同时存在“立即查询/调用”和“异步通知”两种通信需求。

### Decision

采用：

```text
同步能力 / 当前状态
        → C++ Call

异步状态 / 执行结果 / 事件
        → eventcpp
```

### Rationale

避免所有调用都被强制转换成异步消息，也避免所有状态变化都形成强耦合直接调用。

### Consequences

模块之间的通信语义更加明确。

---

# 30. 架构演进原则

后续架构演进应遵循：

1. 先确认职责，再设计接口。
2. 先确认模块边界，再设计类。
3. 先确认数据流，再定义消息。
4. 先确认并发需求，再确定线程。
5. 不因为实现方便而破坏模块边界。
6. 不因为单个功能需求而将外部依赖泄漏到业务层。
7. 不将暂时实现细节提前固化为架构约束。
8. 架构发生重大变化时必须同步更新 ADR。
9. 如果实现需要违反本文档中的架构约束，应先修改 ARCHITECTURE.md，再修改代码。
10. SPEC.md 与 ARCHITECTURE.md 必须保持一致：需求变化修改 SPEC，架构变化修改 ARCHITECTURE。

---

# 31. 当前架构总览

最终形成以下职责结构：

```text
                         GCS
                          │
                        Zenoh
                          │
                   Communication
                          │
                          ▼
┌───────────────────────────────────────────────────┐
│                    Core Runtime                   │
├───────────────────────────────────────────────────┤
│                                                   │
│ Task Scheduler                                    │
│      │                                            │
│      ▼                                            │
│ Task Execution Engine                             │
│      │                                            │
│      ├──── BehaviorTree.CPP                       │
│      │          │                                │
│      │          └── Nodes                         │
│      │                  │                         │
│      │        ┌─────────┼──────────┐              │
│      │        ▼         ▼          ▼              │
│      │   Flight     Perception   Target           │
│      │   Control                  Mgmt            │
│      │        │         │          │              │
│      │        ▼         ▼          ▼              │
│      │       PX4      Plugins     Target          │
│      │                 │                          │
│      │           ┌─────┴─────┐                    │
│      │           ▼           ▼                    │
│      │       Detection    Tracking                │
│      │                                               │
│ UAV State Management ───── StateMonitor            │
│                                                   │
│ Perception ──────── Video Streaming ── ZLMediaKit │
│                                                   │
│ Plugin Management ─────────────── Pluma            │
│                                                   │
│ Geographic Tool ───────────── GeographicLib        │
│                                                   │
└───────────────────────────────────────────────────┘

通信原则：

C++        → 同步能力调用 / 当前状态查询
eventcpp   → 进程内异步事件
Zenoh      → 网络业务通信
```

本架构的核心思想是：

> **Task 表达“做什么”，Task Execution Engine 决定“怎么做”，BehaviorTree 负责“如何执行”，Capability Module 提供“能做什么”，Adapter/Plugin 负责“具体怎么接入”。**

这四层职责保持稳定后，后续具体类、接口、线程、消息和实现技术可以在不破坏总体架构的情况下逐步细化。