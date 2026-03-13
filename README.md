# 🪐 Stars 陪伴系统

---

## 一、概述

### 1.1 项目背景

Stars 是一个 **基于 LLM + 3D 角色驱动的实时交互陪伴系统**，目标是让角色具备“能聊、会记、会主动、可扩展”的长期交互能力。  

系统采用 **前后端分离** 设计：  
- **前端**：JavaScript ES Modules + Three.js + Socket.io，负责渲染、交互和语音联动  
- **后端**：Python Flask + 调度任务，负责对话、记忆、Todo、Heartbeat、工具与推送链路  

核心技术栈：  
- **模型与语音**：OpenAI 兼容 LLM 接口、VolcEngine TTS、音素时间轴口型同步  
- **工具体系**：本地 Tool Registry + MCP Client（stdio）  
- **推送能力**：Service Worker + Web Push 离线触达  

### 1.2 项目特色

✅ **口型精确同步**：基于音素级别的对齐，精确到毫秒  
✅ **表情动作系统**：LLM驱动的表情动作标签，支持眉眼口独立控制 + lerp平滑过渡  
✅ **多维情感表达**：心情、目标实时展示，支持日记本、计划本、支持heartbeat主动发消息  
✅ **完整的对话链路**：支持流式输出，可中断/重试/回溯  
✅ **长短期分层记忆**：用户/角色画像 + 月/周/日历史摘要 + 当前上下文  
✅ **可扩展工具箱**：支持本地工具调用与 MCP Server 扩展  
✅ **多模型 Agentic**：统一 LLM 服务切换模型，支持 ReAct/工具调用流程  

### 1.3 效果演示
#### 1.3.1 基础对话功能，支持多种角色形象
![心情与目标](./demo/demo.png)

#### 1.3.2 日常聊天中，ta会将重要事项记入计划本
![日常关心与计划本](./demo/todo.png)

#### 1.3.3 ta可以主动搜索用户可能感兴趣的话题引入
![主动搜索功能](./demo/share.png)

#### 1.3.4 ta会主动关心并问询，离线推送到用户手机上
![主动心跳包与离线推送](./demo/heartbeat.png)

#### 1.3.5 随着对话的深入，ta会觉醒自己的人格和世界观
![深度对话](./demo/talk.png)

---

## 二、系统架构

```
AI-Agent/
├── index.html                              # 前端入口页面
├── sw.js                                   # Service Worker（Web Push）
├── configure.json                          # 全局运行配置
├── mcpServers.json                         # MCP Server 配置
│
├── configure/
│   ├── morph_map.json                      # 表情映射表
│   └── motion_map.json                     # 动作映射表
│
├── js/
│   ├── main.js                             # 启动与资源加载入口
│   ├── login.js                            # 登录与用户初始化
│   ├── app-settings.js                     # 设置面板与配置保存
│   ├── user-context.js                     # 当前 user_id 管理
│   ├── chat-service.js                     # 聊天流式链路
│   ├── ui-manager.js                       # UI、状态栏、Todo/日记弹层
│   ├── voice-sync.js                       # TTS 队列与口型同步
│   ├── character-driver.js                 # 表情/动作驱动
│   ├── three-engine.js                     # Three.js 渲染循环
│   ├── speech.js                           # 浏览器语音识别
│   ├── push.js                             # Web Push 前端逻辑
│   ├── config.js                           # 前端配置中心
│   └── vendor/
│       ├── three.module.js
│       ├── ammo.js
│       └── socket.io.min.js
│
├── css/                                    # 前端样式
├── demo/                                   # README 演示资源
├── models/                                 # 模型与动作资源
│
└── backend/
    ├── app.py                              # 后端主入口（Flask）
    ├── requirements.txt                    # 后端依赖
    ├── Dockerfile                          # 后端容器部署
    ├── prompts/                            # 提示词模板与角色配置
    ├── volcengine                          # TTS SDK 相关
    ├── memory                              # 用户记忆数据根目录
    └── logic/
        ├── scheduler_jobs.py               # 定时任务调度
        ├── chat_context.py                 # 对话上下文拼装
        ├── ltm_schema.py                   # 长期记忆结构
        ├── user_profile.py                 # 用户档案读写
        ├── agents/                         # 模型代理与工具注册
        │   ├── openai_compatible_agent.py
        │   ├── agentic_llm_wrapper.py
        │   └── tool_registry.py
        ├── services/                       # 业务服务层
        │   ├── llm_service.py
        │   ├── heartbeat_service.py
        │   ├── memory_service.py
        │   ├── todo_service.py
        │   ├── search_service.py
        │   ├── push_service.py
        │   ├── mcp_service.py
        │   ├── tts_service.py
        │   └── faiss_service.py
        └── utils/                          # 工具层
            ├── weather_utils.py
            ├── store_utils.py
            ├── path_utils.py
            └── timezone_utils.py
```

--- 

# 三、功能描述

### 3.1 前端

#### 3.1.1 启动与登录链路

1. 初始化：加载配置文件、初始化 Three.js 场景。
2. 用户登录： 用户创建/校验、保存登录信息。
3. 进入会话前：历史记录载入、Socket 注册、推送开关绑定、位置写入。

#### 3.1.2 聊天主链路

1. 消息发送与流式接收：`POST /chat_stream`
2. 流式解析回复文本内容，支持情绪/动作/心情/目标标签解耦；将回复切段送入 TTS 生成队列。  
3. 在进入会话时拉取历史；支持重试、编辑回溯、手动停止。  

#### 3.1.3 LLM表情动作驱动

1. LLM输出表情动作标签，如：
``` [眉毛:眉毛抬高:强，眼睛:眯眼笑:强，嘴巴:嘴角上扬:强，动作:介绍:强]  ```
2. 表情采用时间轴分段演进策略，根据 TTS 语音播放的进度，经历“全量呈现 -> 局部演变 -> 随机回归”三个阶段：
   - 全量呈现期：完全执行 LLM 指令。眉毛、眼睛、嘴巴均按指令强度 100% 表现。
    - 演变过渡期： （1）眼睛：平滑取消 LLM 指令，恢复至自然状态并仅保留“生理性随机眨眼”。 （2）眉毛与嘴巴：维持 LLM 指令权重，但注入基于正弦波的微频率扰动，模拟肌肉微颤。
   - 随机闲置期 (语音结束)：LLM 指令淡出，系统接管随机闲置表情和常规眨眼逻辑。
3. 播放对应动作vmd文件，采用lerp平滑淡入淡出。

#### 3.1.4 口型对齐

1. 调用后端tts服务：
   - 调用 TTS API 生成音频文件 wav
   - 采用 pypinin 库提取汉字拼音序列（"你好" → "ni hao"）
   - 调用 TorchFA 强制对齐，生成音素时间轴，保存至 TextGrid
   - 采用 socketio 通知前端任务完成
2. 前端驱动mmd完成表演：
   - 通过 Socket 回传，拉取音频与 TextGrid
   - 根据拼音映射到mmd口型 ```a, e, i, o, u```
   - 按 `seqId` 串行播放音频，保证多段回复顺序一致；播放阶段同步更新口型、表情、动作、心情与目标。 

#### 3.1.5 UI 交互链路

1. 聊天气泡与内容渲染：统一管理 Markdown、链接卡片、时间戳、重播/重新生成功能。
2. 状态栏与弹窗：提供在线/思考/说话状态、心情目标、好感度显示；管理日记弹窗与 Todo 弹窗，支持 Todo 的新增、编辑、勾选、删除与持久化。
3. 输入与设备适配：处理键盘适配、输入框自适应高度、移动端视口锁定等。

#### 3.1.6 Three.js 渲染与驱动循环

1. 渲染基础设施：负责场景、相机、光照、渲染器、轨道控制器初始化。  
2. 动画驱动循环：在动画循环中持续驱动角色表情、眨眼、动作更新与画面渲染。
3. 视口适配：提供窗口尺寸变化适配，降低移动端键盘弹起时的画面畸变。  

#### 3.1.7 Web Push 链路

1. 能力检测与状态管理：前端持久化推送开关状态并检测浏览器能力（Service Worker / PushManager / 权限）。
2. 启用/关闭推送：开启推送时注册 SW、申请权限、订阅并上报后端。 关闭推送时取消订阅并上报后端清理 endpoint。  

#### 3.1.8 配置与个性化链路

- `config.js` 统一管理前端全局配置、后端地址、动作/表情映射。
- 支持模型切换、服装切换、场景切换，保存后即时刷新角色与舞台；  
- 支持自动播放 TTS、长期记忆下载/上传（文件夹或 zip）和推送开关联动。  

### 3.2 后端

#### 3.3.1 记忆压缩

1. 触发条件：非 system 消息轮次达到 20 轮以上 & 当前上下文字数达到 2000 字以上。  
2. 触发后行为：后台异步执行记忆整理，不阻塞当前对话返回；产出并更新日记、今日摘要、历史摘要、画像增量、待办增量；标记已完成压缩的用户信息位置。
3. 摘要替换上下文逻辑：后续 prompt 组装时，不再依赖整段长历史，而是优先注入“长期记忆 + 摘要记忆”；前端同步最近少量对话窗口，保持连续感并控制上下文膨胀。  

#### 3.3.2 长短期分层记忆

1. 记忆结构  
- 长期记忆：用户画像和角色画像（性格/偏好/日常习惯，滚动更新）、角色记事本
- 中长期记忆：分层历史摘要（月/周/日历史摘要）
- 短期记忆：当前窗口的上下文

2. 对话注入方式：每轮对话组装 system prompt 时注入角色设定、用户画像、角色画像、历史摘要、今日摘要、记事本与交互要求。  

#### 3.3.3 记事本：提醒 / 搜索 / 记录 功能链路

1. 提醒（Reminder）：设置cron定时任务，支持离线推送
2. 搜索（Search）：调用搜索引擎返回十条结果，再由 LLM 选择一条推送给用户。
3. 记录（Record）： 将任务内容写入长期记忆，作为后续对话的引用信息。  

#### 3.3.4 Heartbeat 主动触达

1. 触发条件：每分钟检查一次；凌晨 00:00-08:00 不触发；用户最近活跃在“昨天/今天”；距离上次用户发消息过去了n小时的整数倍时间（n可设置）。
2. 触发流程：询问LLM是否要主动发送消息，若为空/`SILENT`则不推送、不处理；若有有效内容，则触发 Web Push 推送给用户。

#### 3.3.5 本地工具与 MCP Client 

1. 本地工具：通过统一注册表声明工具名、描述、参数与处理函数；
2. MCP Client：支持从`mcpServers.json`加载配置；支持 `jsonl` 和 `content-length` 两种 stdio 协议；支持 server 级自检、超时控制、stderr 采集与错误隔离（单个 server 失败不影响主流程）。
3. 工具调用流程：Agentic 模式下，模型先判断是否调用工具，如果是，则由统一 handler 转发到对应 server，完成工具执行返回给模型。

### 3.3 Token消耗与推理费用优化

为降低LLM实际部署中的token消耗与API费用，从以下三方面进行系统性优化，优化后，LLM调用的API成本降低约66%： 
1. Prompt Caching & KV Cache Hit Rate 最大化：通过语义缓存与精确前缀匹配，将重复上下文复用率提升至75%-85%。
2. Prompt Compression：去除无关冗余信息，并采用上下文摘要聚合方式，压缩历史对话长度。
3. System Prompt Placement Optimization：将时间戳等动态变量从system prompt前置移至独立后置，避免破坏KV缓存前缀一致性，实现更高复用率。
