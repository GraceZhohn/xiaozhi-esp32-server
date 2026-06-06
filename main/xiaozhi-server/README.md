# xiaozhi-esp32-server

小智 ESP32 语音助手服务端，为 ESP32 硬件设备提供 WebSocket 语音交互服务。

核心流程：**设备发送音频 → 服务端语音识别(ASR) → 大模型(LLM)生成回复 → 语音合成(TTS) → 音频流回传设备播放**

## 目录结构

```
xiaozhi-server/
├── app.py                    # 程序入口，启动 WebSocket + HTTP 服务
├── config.yaml               # 默认配置文件
├── docker-compose.yml        # Docker 部署文件
├── requirements.txt          # Python 依赖
├── agent-base-prompt.txt     # Agent 基础提示词
│
├── config/                   # 配置与日志
│   ├── assets/               # 静态音频资源（唤醒词、提示音、绑定码等）
│   ├── config_loader.py      # 配置加载器（支持远程 API 拉取）
│   ├── logger.py             # 日志配置（基于 loguru）
│   ├── manage_api_client.py  # 智控台 API 客户端
│   └── settings.py           # 全局设置
│
├── core/                     # 核心业务逻辑
│   ├── connection.py         # 连接处理器（核心枢纽）
│   ├── websocket_server.py   # WebSocket 服务器
│   ├── http_server.py        # HTTP 服务器（OTA / 视觉分析）
│   ├── auth.py               # JWT 认证
│   ├── api/                  # HTTP API 接口（OTA、视觉分析）
│   ├── handle/               # 消息处理器
│   ├── providers/            # 各类 AI 服务提供者（插件化）
│   └── utils/                # 工具类
│
├── models/                   # 本地模型文件
│   ├── SenseVoiceSmall/      # 阿里 SenseVoice 语音识别模型
│   └── snakers4_silero-vad/  # Silero VAD 语音活动检测模型
│
├── plugins_func/             # 插件系统
│   ├── functions/            # 具体插件实现
│   ├── loadplugins.py        # 插件自动加载
│   └── register.py           # 插件注册机制
│
└── performance_tester/       # 性能测试工具
```

## 核心架构

### 启动流程

```
app.py → main()
  ├── load_config()            # 加载配置（优先 data/.config.yaml → config.yaml）
  ├── WebSocketServer(config)  # 初始化全局模块（VAD/ASR/LLM/TTS/Memory/Intent）
  ├── SimpleHttpServer(config) # 初始化 HTTP 服务（OTA/视觉分析）
  └── asyncio.run()            # 并发启动两个服务
```

VAD、ASR、LLM、Intent、Memory 等模块在服务启动时全局初始化一次，所有连接共享实例；TTS 是每个连接单独初始化的。

### 连接处理器（core/connection.py）

`ConnectionHandler` 是整个系统的核心枢纽，每个 WebSocket 连接创建一个实例，管理一次完整的对话会话：

| 属性 | 说明 |
|------|------|
| vad / asr / tts / llm / memory / intent | 依赖的 AI 组件 |
| client_audio_buffer / client_have_voice / client_voice_stop | 音频状态 |
| dialogue | 对话管理（Dialogue 对象） |
| sentence_id | 当前句子 ID |
| func_handler | 工具调用处理器（UnifiedToolHandler） |
| stop_event / executor / report_queue | 线程与任务管理 |
| headers / device_id / sample_rate / audio_format | 设备信息 |

### 消息处理流程（core/handle/）

```
设备 WebSocket 消息
  │
  ├── 音频消息 → receiveAudioHandle.py
  │     ├── VAD 检测（是否有人说话）
  │     ├── ASR 语音识别（音频 → 文字）
  │     └── startToChat() → 进入对话流程
  │
  ├── 文本消息 → textHandle.py → TextMessageProcessor
  │     ├── helloMessageHandler.py    # 握手 / 初始化
  │     ├── listenMessageHandler.py   # 监听模式控制
  │     ├── iotMessageHandler.py      # IoT 设备描述
  │     ├── mcpMessageHandler.py      # MCP 工具协议
  │     ├── abortMessageHandler.py    # 打断处理
  │     ├── pingMessageHandler.py     # 心跳
  │     └── serverMessageHandler.py   # 服务端消息
  │
  └── 发送音频 → sendAudioHandle.py
        ├── sendAudioMessage()  # 发送 TTS 音频给设备
        ├── send_stt_message()  # 发送 ASR 识别结果
        └── send_tts_message()  # 发送 TTS 状态消息
```

### 完整对话数据流

```
┌─────────┐    WebSocket     ┌──────────────────┐
│  ESP32  │ ◄──────────────► │  WebSocketServer │
│  设备   │   音频/文本消息   │                  │
└─────────┘                  └────────┬─────────┘
                                      │ 每个连接创建
                                      ▼
                             ┌──────────────────┐
                             │ ConnectionHandler│
                             └────────┬─────────┘
                                      │
    ┌─────────────────────────────────┼─────────────────────────────────┐
    │                                 │                                 │
    ▼                                 ▼                                 ▼
┌────────┐  音频流   ┌────────┐  文字   ┌────────┐  文字   ┌────────┐
│  VAD   │──────────►│  ASR   │────────►│  LLM   │────────►│  TTS   │
│语音检测 │          │语音识别 │         │大模型   │         │语音合成 │
└────────┘          └────────┘         └───┬────┘         └───┬────┘
                                          │                   │
                                          │ 工具调用           │ Opus 音频
                                          ▼                   ▼
                                   ┌────────────┐     ┌──────────────┐
                                   │ ToolHandler │     │ tts_audio_   │
                                   │ IoT/MCP/    │     │ queue        │
                                   │ Plugin      │     └──────┬───────┘
                                   └────────────┘            │
                                                             ▼
                                                    ┌──────────────┐
                                                    │ 音频播放线程  │
                                                    │ → WebSocket  │
                                                    │ → ESP32 设备 │
                                                    └──────────────┘
```

## Providers 架构（core/providers/）

整个项目最核心的设计——插件化的 AI 服务提供者，每个类别都有基类和多种实现。

### ASR（语音识别）

| 文件 | 说明 |
|------|------|
| base.py | 基类 |
| aliyun_stream.py | 阿里云流式 |
| aliyunbl_stream.py | 阿里云百炼流式 |
| doubao_stream.py | 豆包流式 |
| xunfei_stream.py | 讯飞流式 |
| sherpa_onnx_local.py | 本地 Sherpa-ONNX |
| fun_local.py | FunASR 本地 |
| fun_server.py | FunASR 服务端 |
| openai.py | OpenAI 兼容接口 |
| qwen3_asr_flash.py | 通义千问 ASR |
| baidu.py | 百度语音识别 |
| tencent.py | 腾讯语音识别 |
| vosk.py | Vosk 本地识别 |

### TTS（语音合成）

| 文件 | 说明 |
|------|------|
| base.py | 基类（含音频播放线程、文本分段逻辑） |
| dto/dto.py | 数据传输对象（SentenceType / ContentType / InterfaceType） |
| minimax_httpstream.py | MiniMax HTTP 流式 |
| alibl_stream.py | 阿里百炼流式 |
| aliyun_stream.py | 阿里云流式 |
| huoshan_double_stream.py | 火山引擎双流 |
| index_stream.py | IndexTTS 流式 |
| xunfei_stream.py | 讯飞流式 |
| edge.py | Edge TTS |
| fishspeech.py | FishSpeech |
| gpt_sovits_v2.py | GPT-SoVITS v2 |
| gpt_sovits_v3.py | GPT-SoVITS v3 |
| cozecn.py | Coze CN |
| openai.py | OpenAI 兼容接口 |
| siliconflow.py | SiliconFlow |
| tencent.py | 腾讯语音合成 |
| doubao.py | 豆包语音合成 |
| custom.py | 自定义 TTS |
| paddle_speech.py | PaddleSpeech 本地 |

### LLM（大语言模型）

| 文件 | 说明 |
|------|------|
| base.py | 基类（response / response_with_functions） |
| openai/ | OpenAI 兼容（最通用） |
| ollama/ | Ollama 本地部署 |
| gemini/ | Google Gemini |
| AliBL/ | 阿里百炼 |
| coze/ | Coze |
| dify/ | Dify |
| fastgpt/ | FastGPT |
| homeassistant/ | Home Assistant |
| xinference/ | Xinference |

### 其他 Providers

| 目录 | 说明 |
|------|------|
| vad/ | 语音活动检测（Silero VAD） |
| intent/ | 意图识别（function_call / intent_llm / nointent） |
| memory/ | 记忆管理（mem0ai / mem_local_short / powermem / nomem） |
| vllm/ | 视觉语言模型（OpenAI 兼容，用于视觉分析） |

## 工具系统（core/providers/tools/）

```
tools/
├── base/                    # 工具基础设施
│   ├── tool_executor.py     # 工具执行器基类
│   └── tool_types.py        # 工具类型枚举
├── device_iot/              # 设备端 IoT 控制
├── device_mcp/              # 设备端 MCP 协议
├── server_mcp/              # 服务端 MCP 服务
├── mcp_endpoint/            # MCP 端点
├── server_plugins/          # 服务端插件执行器
├── unified_tool_handler.py  # 统一工具调度器
└── unified_tool_manager.py  # 统一工具管理器
```

工具类型：

- **SERVER_PLUGIN** — 服务端插件（天气、新闻、音乐等）
- **SERVER_MCP** — 服务端 MCP 工具
- **DEVICE_IOT** — 设备端 IoT 控制
- **DEVICE_MCP** — 设备端 MCP 工具
- **MCP_ENDPOINT** — MCP 端点工具

## 插件系统（plugins_func/）

```
plugins_func/
├── register.py              # Action / ActionResponse 注册机制
├── loadplugins.py           # 自动扫描加载
└── functions/               # 具体插件
    ├── get_weather.py       # 天气查询
    ├── get_time.py          # 时间查询
    ├── get_news_from_chinanews.py  # 中新网新闻
    ├── get_news_from_newsnow.py    # NewsNow 新闻
    ├── web_search.py        # 联网搜索
    ├── hass_*.py            # Home Assistant 控制
    ├── play_music.py        # 音乐播放
    ├── change_role.py       # 角色切换
    ├── search_from_ragflow.py # RAGFlow 知识库搜索
    └── handle_exit_intent.py  # 退出意图处理
```

## TTS 数据流详解

以 `minimax_httpstream.py` 为例，TTS 的完整数据流：

```
LLM 文字输出
  → tts_text_queue（文本队列）
  → tts_text_priority_thread（消费线程，按标点分段）
  → to_tts_single_stream（清理 Markdown、替换词校正）
  → text_to_speak（异步 API 请求）
  → PCM buffer（hex 解码 API 响应）
  → opus_encoder（PCM → Opus 编码）
  → handle_opus → tts_audio_queue
  → _audio_play_priority_thread（消费队列）
  → sendAudioMessage → WebSocket → 设备
```

TTS 消息类型（SentenceType）：

| 类型 | 说明 |
|------|------|
| FIRST | 句子开始，携带文本内容 |
| MIDDLE | 句子中间，携带 Opus 音频数据 |
| LAST | 句子结束，触发停止播放 |

## 配置说明

系统支持两级配置：

1. `config.yaml` — 默认配置
2. `data/.config.yaml` — 用户自定义覆盖配置（优先级更高）

如果使用智控台，所有配置通过远程 API 管理，本地配置文件不生效。

关键配置项：

| 配置 | 说明 |
|------|------|
| server.ip / port | WebSocket 监听地址和端口 |
| server.http_port | HTTP 服务端口（OTA / 视觉分析） |
| log.log_level | 日志等级（INFO / DEBUG） |
| delete_audio | 使用完音频文件后是否删除 |
| close_connection_no_voice_time | 无语音输入超时断开时间（秒） |
| proactive_trigger_time | 用户无交互主动触发大模型时间（秒），默认30秒 |
| tts_timeout | TTS 请求超时时间（秒） |
| exit_commands | 退出对话的命令词 |
| wakeup_words | 唤醒词列表 |
| plugins.* | 各插件配置 |

## 运行环境

- Python 3.10
- 依赖详见 requirements.txt
- 需要 ffmpeg（系统自动检测）
