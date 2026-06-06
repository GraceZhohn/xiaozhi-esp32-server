# 小智数字人模块 (digital-human) 代码目录结构与架构说明

## 项目概述

digital-human 是集成在 xiaozhi-esp32-server 中的数字人测试与交互模块。它提供了一个**基于浏览器的数字人前端界面**（含 Live2D 模型渲染、语音通讯、摄像头预览等能力），同时集成了**本地唤醒词检测运行时**，用于联调数字人的完整交互链路。

---

## 整体架构

```
┌─────────────────────────────────────────────────────┐
│                  浏览器 (Browser)                    │
│  ┌───────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Live2D    │  │ 音频引擎  │  │ 网络通信模块      │  │
│  │ 渲染层    │  │ WebAudio │  │ WebSocket 客户端  │  │
│  │ Pixi.js   │  │ Opus编   │  │ OTA 连接器        │  │
│  │ Cubism 4  │  │ 解码器   │  │ 唤醒词桥接器      │  │
│  └───────────┘  └──────────┘  └──────┬───────────┘  │
│                                      │              │
│                          ┌───────────▼───────────┐  │
│                          │ UI 控制器 & MCP 工具   │  │
│                          └───────────────────────┘  │
└──────────────────────────┬──────────────────────────┘
                           │ HTTP / WebSocket
                           │
┌──────────────────────────▼──────────────────────────┐
│              Python 本地唤醒词运行时                   │
│  ┌──────────┐  ┌────────────┐  ┌──────────────────┐  │
│  │ HTTP     │  │ EventBridge│  │ Plugin System    │  │
│  │ Server   │◄─┤ WebSocket  │──┤ ├─ AudioPlugin   │  │
│  │ :8006    │  │ 发布订阅    │  │ └─ WakeWordPlugin│  │
│  └──────────┘  └────────────┘  └──────┬───────────┘  │
│                                        │              │
│                        ┌───────────────▼───────────┐  │
│                        │  sherpa-onnx 唤醒词检测器  │  │
│                        │  + sounddevice 麦克风采集  │  │
│                        └───────────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

**核心交互流程**：

1. 浏览器通过 `start.py` 启动的 HTTP 服务器加载页面（`index.html`）
2. 前端通过 OTA 协议连接服务器，获取 WebSocket 地址
3. 前端通过 WebSocket 与 AI 服务端进行双向音频/文本通信
4. 本地唤醒词运行时通过麦克风采集音频，由 sherpa-onnx 检测唤醒词
5. 检测到唤醒词后通过 EventBridge（WebSocket）推送到前端，前端自动发起拨号连接

---

## 技术栈

| 层级 | 技术 | 用途 |
|------|------|------|
| **前端渲染** | Pixi.js 8 + Live2D Cubism 4 SDK | 数字人 3D 模型渲染和交互动画 |
| **前端音频** | Web Audio API + Opus (WASM) | 音频采集、编码（Opus）、解码、播放 |
| **前端网络** | 原生 WebSocket + Fetch API | 实时通信、OTA 协议连接、唤醒词桥接 |
| **前端构建** | ES Modules (原生) | 模块化组织，无构建工具依赖 |
| **后端服务** | Python http.server (标准库) | 静态文件服务、WebSocket 升级、HTTP API |
| **后端检测** | sherpa-onnx (1.12) | 唤醒词关键词检测（Zipformer 模型） |
| **后端音频** | sounddevice | 系统麦克风音频采集 |
| **后端辅助** | pypinyin | 唤醒词拼音音素转换 |

---

## 目录结构详解

```
digital-human/
│
├── start.py                          # 【启动入口】Python 模块入口
│                                     # - 加载配置、初始化日志
│                                     # - 启动唤醒词运行时（AudioPlugin + WakeWordPlugin）
│                                     # - 启动 HTTP/WebSocket 服务器（:8006）
│                                     # - 支持运行时热重启
│
├── index.html                        # 【主页面】数字人测试界面
│                                     # - Live2D 画布（全屏模型展示）
│                                     # - 聊天消息流（弹幕样式）
│                                     # - 底部控制栏（设置/摄像头/拨号/录音）
│                                     # - 设置弹窗（设备配置/唤醒词/MCP工具/皮肤切换）
│                                     # - 摄像头预览窗口（可拖动）
│                                     # - 日志面板
│
├── favicon.ico                       # 网站图标
│
├── css/
│   ├── index.css                     # 全局样式：布局、背景、控制栏、设置弹窗、聊天流等
│   └── bg.png                        # 背景纹理图案
│
├── images/                           # 背景图集（3 张可切换的背景图）
│   ├── 1.png
│   ├── 2.png
│   └── 3.png
│
├── js/                               # ====== 前端 JavaScript 模块 ======
│   │
│   ├── app.js                        # 【主应用入口】App 类
│   │                                 # - 初始化 UI 控制器、Opus 编解码器、音频播放器
│   │                                 # - 初始化 MCP 工具、唤醒词桥接器
│   │                                 # - 检测麦克风/摄像头可用性
│   │                                 # - 初始化 Live2D 模型和摄像头
│   │
│   ├── config/
│   │   ├── manager.js                # 【配置管理】loadConfig / saveConfig / getConfig
│   │   │                             # - 从 localStorage 读写设备 MAC、名称、OTA URL 等
│   │   │                             # - 默认唤醒词列表管理
│   │   │                             # - 生成随机 MAC 地址
│   │   └── default-mcp-tools.json    # 默认 MCP 工具定义（JSON 格式）
│   │
│   ├── core/
│   │   ├── audio/
│   │   │   ├── opus-codec.js         # 【Opus 编码器】
│   │   │   │                         # - 封装 Emscripten 编译的 libopus WASM
│   │   │   │                         # - 提供 Opus 编码（VoIP 模式，16kHz，16kbps）
│   │   │   │                         # - checkOpusLoaded / initOpusEncoder
│   │   │   │
│   │   │   ├── player.js             # 【音频播放器】AudioPlayer 类
│   │   │   │                         # - Opus 解码（libopus WASM）
│   │   │   │                         # - Web Audio API 播放调度
│   │   │   │                         # - 可配置缓冲策略（超时 + 批量）
│   │   │   │                         # - 分析器节点（供 Live2D 嘴部动画驱动）
│   │   │   │
│   │   │   ├── recorder.js           # 【音频录制器】AudioRecorder 类
│   │   │   │                         # - 基于 AudioWorklet（优先）/ ScriptProcessor（降级）
│   │   │   │                         # - 实时 PCM 采集 → Opus 编码 → WebSocket 发送
│   │   │   │                         # - checkMicrophoneAvailability 权限检测
│   │   │   │
│   │   │   └── stream-context.js     # 【流式播放上下文】StreamingContext 类
│   │   │                             # - 管理 Opus 解码后的 PCM 队列播放
│   │   │                             # - 支持清空缓冲、获取播放进度
│   │   │                             # - 精确的 Web Audio 调度
│   │   │
│   │   ├── mcp/
│   │   │   └── tools.js              # 【MCP 工具管理】
│   │   │                             # - 加载/编辑/删除 MCP 工具（JSON Schema 格式）
│   │   │                             # - 工具模拟响应配置
│   │   │                             # - 通过 WebSocket 发送 MCP 工具调用
│   │   │
│   │   └── network/
│   │       ├── websocket.js          # 【WebSocket 消息处理】WebSocketHandler 类
│   │       │                         # - hello 握手协议
│   │       │                         # - 消息分发：tts / stt / llm / audio / mcp
│   │       │                         # - 会话管理、Live2D 说话动画触发
│   │       │                         # - 情绪表情提取与传递
│   │       │
│   │       ├── wakeword-bridge.js    # 【唤醒词桥接器】
│   │       │                         # - 连接本地唤醒词运行时的 WebSocket（ws://127.0.0.1:8006/wakeword-ws）
│   │       │                         # - 自动重连机制（指数退避，上限 5s）
│   │       │                         # - 请求/响应（requestId 模式）、消息发布
│   │       │                         # - 接收 wake_word_detected 事件，触发自动拨号
│   │       │
│   │       └── ota-connector.js      # 【OTA 连接器】
│   │                                 # - 向 OTA 服务器 POST 设备信息，获取 WebSocket 地址
│   │                                 # - 组装 WebSocket URL（含 authorization / device-id 参数）
│   │                                 # - 配置校验
│   │
│   ├── live2d/
│   │   ├── live2d.js                 # 【Live2D 管理器】Live2DManager 类
│   │   │                             # - 模型初始化（支持 Hiyori / Natori 双模型）
│   │   │                             # - 嘴部动画（基于音频分析器驱动 / 随机模式）
│   │   │                             # - 点击交互（单击/双击/滑动 → 触发对应动画）
│   │   │                             # - 情绪动作映射（happy/laughing/sad/angry 等）
│   │   │                             # - 模型热切换（无需刷新页面）
│   │   │
│   │   ├── cubism4.min.js            # Live2D Cubism 4 SDK（交互框架）
│   │   ├── live2dcubismcore.min.js   # Live2D Cubism 4 Core（渲染引擎）
│   │   └── pixi.js                   # Pixi.js 8 渲染引擎
│   │
│   ├── ui/
│   │   ├── controller.js             # 【UI 控制器】UIController 类
│   │   │                             # - 控制栏事件（设置/摄像头/拨号/录音按钮）
│   │   │                             # - 聊天消息流管理（添加消息、自动滚动）
│   │   │                             # - 音频可视化器（Canvas 频谱）
│   │   │                             # - 设置弹窗交互（标签页切换、配置保存）
│   │   │                             # - 背景图切换、模型切换
│   │   │                             # - 连接状态、录音状态 UI 更新
│   │   │
│   │   └── background-load.js        # 背景图加载检测
│   │                                 # - 等待背景图加载完成后显示模型加载提示
│   │
│   └── utils/
│       ├── blocking-queue.js         # 【异步阻塞队列】BlockingQueue 类
│       │                             # - 生产者-消费者模式
│       │                             # - 支持批量出队（min 数量 + timeout 超时）
│       │                             # - 用于音频包缓冲和解耦
│       │
│       ├── libopus.js                # Opus 编解码器 WASM（Emscripten 编译）
│       │                             # - 导出 Module 全局对象
│       │                             # - 提供 opus_encoder / opus_decoder C API 封装
│       │
│       └── logger.js                 # 【日志工具】
│                                     # - 带时间戳的日志输出
│                                     # - 四种级别：info / success / warning / error
│                                     # - 同时输出到控制台和页面日志面板
│
├── resources/                        # ====== Live2D 模型资源 ======
│   │
│   ├── hiyori_pro_zh/                # 【日代モデル】Hiyori 模型（中文版）
│   │   ├── ReadMe.txt                # 模型说明
│   │   ├── hiyori_pro_t04.can3       # CAN3 工程文件（Cubism Editor）
│   │   ├── hiyori_pro_t11.cmo3       # CMO3 模型文件
│   │   └── runtime/                  # 运行时加载目录
│   │       ├── hiyori_pro_t11.model3.json  # 模型定义文件（入口）
│   │       ├── hiyori_pro_t11.moc3         # MOC3 网格/变形数据
│   │       ├── hiyori_pro_t11.cdi3.json    # 显示信息（参数/部件定义）
│   │       ├── hiyori_pro_t11.physics3.json # 物理模拟（头发/衣物摆动）
│   │       ├── hiyori_pro_t11.pose3.json    # 姿势分组
│   │       ├── hiyori_pro_t11.2048/         # 纹理贴图（2048px）
│   │       │   ├── texture_00.png
│   │       │   └── texture_01.png
│   │       └── motion/                # 动作动画（10 组）
│   │           ├── hiyori_m01.motion3.json
│   │           ├── ...
│   │           └── hiyori_m10.motion3.json
│   │
│   └── natori_pro_zh/                # 【鳴瀬モデル】Natori 模型（中文版）
│       ├── ReadMe.txt
│       ├── natori_pro_t06.cmo3
│       ├── natori_pro_exp_t03.can3           # 表情 CAN3 工程
│       ├── natori_pro_motions_t03.can3       # 动作 CAN3 工程
│       └── runtime/
│           ├── natori_pro_t06.model3.json
│           ├── natori_pro_t06.moc3
│           ├── natori_pro_t06.cdi3.json
│           ├── natori_pro_t06.physics3.json
│           ├── natori.pose3.json
│           ├── natori_pro_t06.4096/          # 纹理贴图（4096px 更高清）
│           │   └── texture_00.png
│           ├── exp/                           # 面部表情（11 种）
│           │   ├── Normal.exp3.json
│           │   ├── Angry.exp3.json
│           │   ├── Smile.exp3.json
│           │   ├── Sad.exp3.json
│           │   ├── Surprised.exp3.json
│           │   ├── Blushing.exp3.json
│           │   ├── exp_01 ~ exp_05.exp3.json
│           └── motions/                       # 动作动画（8 组）
│               ├── mtn_00 ~ mtn_07.motion3.json
│
└── wakeword_runtime/                 # ====== 唤醒词运行时（Python） ======
    │
    ├── __init__.py
    ├── config.json                   # 运行时配置
    │                                 # - wakeword.enabled: 是否启用唤醒词
    │                                 # - model_dir: 模型目录路径
    │                                 # - audio: 采样率/声道/输入设备
    │                                 # - detector: 检测器参数（线程数/分数/阈值/冷却时间）
    │                                 # - logging: 日志级别/目录/文件名
    │
    ├── requirements.txt              # Python 依赖
    │                                 # - sherpa-onnx==1.12.29（唤醒词检测）
    │                                 # - sounddevice>=0.4.4（音频采集）
    │                                 # - pypinyin==0.55.0（拼音转换）
    │
    ├── bridge/
    │   ├── __init__.py
    │   └── event_bridge.py           # 【事件桥】WakewordEventBridge 类
    │                                 # - WebSocket 发布-订阅模式
    │                                 # - 多客户端管理（queue.Queue 列表）
    │                                 # - 事件发布：service_ready / service_stopping / wake_word_detected / wakeword_config
    │                                 # - 消息构建与广播
    │
    ├── config/
    │   ├── __init__.py
    │   ├── config_loader.py          # 【配置加载器】
    │   │                             # - RuntimeConfig dataclass：全局配置对象
    │   │                             # - 子配置：WakewordSettings / AudioSettings / DetectorSettings / LoggingSettings
    │   │                             # - load_config()：从 JSON 读取 + 验证
    │   │                             # - 从 keywords.txt 解析唤醒词列表（拼音格式）
    │   │
    │   └── logging_setup.py          # 【日志配置】
    │                                 # - 同时输出到控制台和文件
    │                                 # - 自动创建日志目录
    │
    ├── core/
    │   ├── __init__.py
    │   ├── detector.py               # 【唤醒词检测器】WakewordDetector 类
    │   │                             # - sherpa-onnx KeywordSpotter 封装
    │   │                             # - 异步检测循环（后台线程）
    │   │                             # - PCM 音频分块处理 → 解码 → 关键词匹配
    │   │                             # - 冷却时间控制（防重复触发）
    │   │                             # - 错误容错（最多 5 次连续错误后停止）
    │   │
    │   ├── detector_assets.py        # 【检测资源管理】DetectorAssetsBuilder
    │   │                             # - 验证模型文件完整性（encoder/decoder/joiner/tokens）
    │   │                             # - 利用 pypinyin 将唤醒词转写为拼音 Token 格式
    │   │                             # - 生成 keywords.txt（sherpa-onnx 格式）
    │   │
    │   └── microphone.py             # 【麦克风监听器】MicrophoneListener 类
    │                                 # - 基于 sounddevice InputStream 实时采集
    │                                 # - 支持多个 AudioListener 订阅者（观察者模式）
    │                                 # - 可配置采样率/声道/输入设备/块大小
    │
    ├── models/                       # 模型文件目录（sherpa-onnx 格式）
    │   ├── encoder.onnx              # Zipformer 编码器模型
    │   ├── decoder.onnx              # 解码器模型
    │   ├── joiner.onnx               # Joiner 模型
    │   ├── tokens.txt                # Token 词典
    │   └── keywords.txt              # 生成的唤醒词关键词文件（拼音 Token 格式）
    │
    ├── plugins/
    │   ├── __init__.py
    │   ├── base.py                   # 【插件基类】Plugin
    │   │                             # - 定义插件生命周期：setup → start → stop → shutdown
    │   │                             # - name / priority 属性
    │   │
    │   ├── manager.py                # 【插件管理器】PluginManager
    │   │                             # - 注册/查询插件（按 priority 排序）
    │   │                             # - 批量生命周期管理（setup_all / start_all / stop_all / shutdown_all）
    │   │
    │   ├── audio.py                  # 【音频插件】AudioPlugin
    │   │                             # - 创建 MicrophoneListener 实例
    │   │                             # - 开启/停止麦克风采集
    │   │                             # - 优先级 10（最先启动）
    │   │
    │   └── wake_word.py              # 【唤醒词插件】WakeWordPlugin
    │   │                             # - 创建 WakewordDetector 实例
    │   │                             # - 从 AudioPlugin 获取音频源
    │   │                             # - 检测到唤醒词后回调 EventBridge 推送
    │   │                             # - 优先级 30
    │   │
    │   └── __init__.py
    │
    └── runtime/
        ├── __init__.py
        ├── app.py                    # 【运行时应用】TestRuntimeApplication
        │                             # - 插件编排：创建 PluginManager，注册 AudioPlugin + WakeWordPlugin
        │                             # - 管理应用启动/停止/重启生命周期
        │                             # - 唤醒词检测回调 → EventBridge 发布
        │
        └── http_server.py            # 【HTTP/WebSocket 服务器】TestRuntimeHttpServer
                                      # - 基于 Python http.server + threading
                                      # - 静态文件服务（当前目录）
                                      # - WebSocket 握手升级（/wakeword-ws）
                                      # - 健康检查端点（/health）
                                      # - 唤醒词配置读写（/wakeword-ws 内协议）
                                      # - 运行时热重启支持
```

---

## 各部分代码职责速查

| 文件 | 职责 |
|------|------|
| `start.py` | 模块启动入口：加载配置 → 启动运行时 → 启动 HTTP 服务器 |
| `index.html` | 数字人测试页面入口：全量 UI 结构定义 |
| `js/app.js` | 前端应用初始化编排：协调各子模块启动顺序 |
| `js/core/audio/opus-codec.js` | Opus 编码器 WASM 封装：PCM → Opus |
| `js/core/audio/player.js` | Opus 解码 + Web Audio 播放：Opus → PCM → 扬声器 |
| `js/core/audio/recorder.js` | 麦克风录音 + Opus 编码 + WebSocket 发送 |
| `js/core/audio/stream-context.js` | 流式播放缓冲/解码/调度引擎 |
| `js/core/network/websocket.js` | AI 服务端 WebSocket 通信：协议握手、消息分发 |
| `js/core/network/wakeword-bridge.js` | 本地唤醒词运行时桥接：WebSocket 连接、自动重连 |
| `js/core/network/ota-connector.js` | OTA 协议客户端：获取服务器 WebSocket 地址 |
| `js/core/mcp/tools.js` | MCP 工具管理：CRUD、模拟响应、WebSocket 集成 |
| `js/live2d/live2d.js` | Live2D 模型管理：加载、嘴部动画、交互事件、情绪映射 |
| `js/ui/controller.js` | UI 状态管理：按钮事件、聊天流、可视化器、设置弹窗 |
| `js/config/manager.js` | 前端配置持久化：localStorage 读写 |
| `js/utils/blocking-queue.js` | 异步阻塞队列：生产者-消费者音频缓冲 |
| `js/utils/logger.js` | 前端日志工具 |
| `wakeword_runtime/runtime/http_server.py` | Python HTTP 服务器：静态文件、WebSocket 升级、健康检查 |
| `wakeword_runtime/runtime/app.py` | 运行时应用编排：插件生命周期管理 |
| `wakeword_runtime/bridge/event_bridge.py` | WebSocket 事件桥：发布-订阅消息通道 |
| `wakeword_runtime/core/detector.py` | sherpa-onnx 唤醒词检测：异步检测循环 |
| `wakeword_runtime/core/detector_assets.py` | 检测资源准备：模型验证、拼音 Token 生成 |
| `wakeword_runtime/core/microphone.py` | sounddevice 麦克风采集：观察者模式分发 |
| `wakeword_runtime/config/config_loader.py` | 运行时配置加载与验证 |
| `wakeword_runtime/plugins/audio.py` | 音频插件：麦克风采集生命周期 |
| `wakeword_runtime/plugins/wake_word.py` | 唤醒词插件：检测器生命周期 + 事件回调 |
| `wakeword_runtime/plugins/manager.py` | 插件管理器：注册/排序/批量生命周期 |
| `wakeword_runtime/plugins/base.py` | 插件基类：定义生命周期接口 |