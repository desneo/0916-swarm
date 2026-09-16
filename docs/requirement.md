本文档描述客户输入的项目背景信息和技术要求：
1. 项目背景与目标
构建一个基于C++17规范的无人机机载智能程序，建议命名空间plane
运行环境：系统ubuntu22.04,支持X64和arm64架构;硬件是基于3588芯片的智能板，通过串口会网口与飞控通信
使用场景：控制己方基于px4或apm的火箭穿越机无人机进行自杀式反制其他无人机

架构要求：
1.使用eventcpp实现的pub/sub总线架构

2.建议使用到的三方库：
MAVSDK:  (飞控串口及 MAVLink 协议封装)
spdlog:  (高性能异步日志)
fmt:  (现代 C++ 占位符字符串格式化)
nlohmann_json:  (JSON 解析与序列化核心)
Zenohcpp, 用于与地面站通信
OpenCV: 4.6.0 (视觉侦察取帧剪裁、相机标定、2D/3D矩阵解算)
GeographicLib: 
magic_enum: enum
Behaviortree: 行为树框架
eventcpp: 自带源码库集成 (进程内多线程安全的 Pub/Sub 事件总线)
Pluma: 轻量化 C++ 跨平台动态库 *.so 加载框架,用于加载算法插件
zlmediakit: 用于推流

3.从外部引入的第三方的算法插件：目标检测、单目标跟踪、图像导引
算法插件使用plugin方式进行封装

建议的模块划分：
1.MessageLink模块：基于zenohcpp实现与地面站及其他飞机的无线通信，核心收发 API: 包含广播至地面站 sendToStation；机间群发 sendToPlanes
序列化机制: 从 MessageLink 接受的底层载荷为通用的 std::string。根据messagetyp字段将内容反序列化为nlohmann::json-->class 对象，放进消息总线分发给具体业务模块。
2.messagcenter:使用eventcpp 实现内部消息总线，项目任何同级模块（例如：飞控模块不依赖目标管理模块）严禁互相交叉引入头文件。业务联动均注册唯一事件号（Event ID），采用“发布(Publish) / 注册回调(Subscribe)”形式彻底解耦。
StateManager: 存储各架无人机以及地面站状态上下文（空速、电池余量、当前经纬度、MAVLink 链路状态）。
TaskManager: 将下发的战术情报组装成可以排队执行的子任务队列，最后分发到行为树执行
StateMonitor: 定时上报飞机以及当前正在执行的任务状态
TargetManager: 汇集并平滑视觉和雷达给出的 SOT/MOT 目标经纬度、置信度及历史轨迹。
SceneManager (场景管理器): 管理默认的飞行安全高度、盘旋半径、禁飞区/任务包线区域等全局上下文环节配置。
SceneMonitor (场景监控器): 在后台线程轮询监控飞机安全底线（电池突降、姿态异常）及集群链路质量。异常时发布高优先级的 Fallback 事件（如返航或接管友机丢包任务）。
TreeExecutor: 行为树执行模块，比如反无任务由解锁 + 起飞 + 导引 + 目标搜索等多个node组成
ImageHandler: 处理图像相关，根据标志位执行多目标还是单目标跟踪，读取图像(从mipi接口)、检测、多目标跟踪、单目标跟踪
FlyControl： 连接飞控，负责无人机的具体操作下发，比如起飞 航线 动作等
camaraControl： 负责相机建联及状态信息获取，指令下发