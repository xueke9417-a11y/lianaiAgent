# 鱼皮 AI 超级智能体应用平台 — 项目全览

## 一、项目概述

这是一个 **AI 智能体应用平台**，由鱼皮（程序员鱼皮）开源，包含两个核心应用：

| 应用 | 说明 | 技术特点 |
|------|------|----------|
| 🤖 **AI 超级智能体（YuManus）** | 全能型 AI 助手，能自主规划、调用工具解决复杂任务 | ReAct Agent + 工具调用 |
| ❤️ **AI 恋爱大师（LoveApp）** | 情感顾问，基于知识库回答恋爱问题 | RAG 检索增强 + 对话记忆 |

项目分为 **后端（Java Spring Boot）** 和 **前端（Vue 3）** 两部分，前后端通过 SSE（Server-Sent Events）流式通信。

---

## 二、技术架构总览

```
┌─────────────────────────────────────────────────────┐
│                     前端 (Vue 3)                     │
│  端口: 3000                                         │
│                                                      │
│  Home.vue ──→ LoveMaster.vue ──→ ChatRoom.vue       │
│           ──→ SuperAgent.vue  ──→ ChatRoom.vue       │
│                                                      │
│  SSE 连接:                                           │
│    /api/ai/love_app/chat/sse  (恋爱大师)             │
│    /api/ai/manus/chat         (超级智能体)            │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP SSE
┌──────────────────────┴──────────────────────────────┐
│                   后端 (Spring Boot 3.4.4)            │
│  端口: 8123, context-path: /api                      │
│                                                      │
│  ┌─────────────────────────────────────────────┐     │
│  │         Controller 层 (AiController)         │     │
│  │  /love_app/chat/sse    /manus/chat           │     │
│  └──────────┬──────────────────┬────────────────┘     │
│             │                  │                       │
│  ┌──────────▼────────┐  ┌─────▼──────────────────┐   │
│  │      LoveApp      │  │  YuManus (ReAct Agent) │   │
│  │                    │  │                         │   │
│  │ • 对话记忆(20条消息)│  │ • ReAct 循环 (think →  │   │
│  │ • RAG 知识库检索   │  │   act → 重复, 最大20步) │   │
│  │ • 结构化输出       │  │ • 工具自主选择与调用    │   │
│  │ • Query 重写       │  │ • 终止工具控制结束      │   │
│  └──────────┬────────┘  └─────┬──────────────────┘   │
│             │                 │                        │
│  ┌──────────▼─────────────────▼──────────────────┐    │
│  │           大模型层 (Spring AI)                 │    │
│  │  DashScope (qwen-plus)  /  Ollama (gemma3:1b) │    │
│  └───────────────────────────────────────────────┘    │
│                                                      │
│  ┌──────────────────────────────────────────────┐     │
│  │              工具集 (ToolCallbacks)            │     │
│  │  WebSearch │ WebScraping │ FileOp │ Terminal  │     │
│  │  ResourceDownload │ PDFGeneration │ Terminate │     │
│  └──────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

---

### 2.1 一分钟看懂两条主链路

```text
恋爱大师：
LoveMaster.vue
  → api/index.js 创建 EventSource
  → AiController.doChatWithLoveAppSSE()
  → LoveApp.doChatByStream()
  → Spring AI ChatClient
  → DashScope 大模型
  → Flux/SSE 持续返回文本
  → 前端拼接到同一条 AI 消息

超级智能体：
SuperAgent.vue
  → api/index.js 创建 EventSource
  → AiController.doChatWithManus()
  → 新建 YuManus 并调用 runStream()
  → BaseAgent 执行 ReAct 循环
  → ToolCallAgent 执行模型请求的工具
  → SseEmitter 按步骤推送结果
  → 前端拆分为多个消息气泡
```

> 注意：当前恋爱大师页面调用的是基础流式聊天接口，不会自动启用 `doChatWithRag()`、工具调用或 MCP。

---

## 三、后端详解

### 3.1 核心模块

```
src/main/java/com/yupi/yuaiagent/
├── YuAiAgentApplication.java          # 启动类（排除 DataSource，简化部署）
├── agent/                              # 🤖 智能体模块
│   ├── BaseAgent.java                  # 智能体抽象基类
│   ├── ReActAgent.java                 # ReAct 循环：think → act → 重复
│   ├── ToolCallAgent.java              # 工具调用代理（实现 think/act）
│   ├── YuManus.java                    # 超级智能体（配置系统提示词和参数）
│   └── model/AgentState.java           # 状态枚举：IDLE / RUNNING / FINISHED
├── app/                                # 🏠 应用模块
│   └── LoveApp.java                    # 恋爱大师（基础聊天 + 可选 RAG/工具/MCP）
├── controller/                         # 🛣️ API 控制器
│   ├── AiController.java               # 核心 AI 接口
│   └── HealthController.java           # 健康检查
├── tools/                              # 🔧 工具集
│   ├── ToolRegistration.java           # 工具注册中心 (7个工具)
│   ├── WebSearchTool.java              # 网络搜索
│   ├── WebScrapingTool.java            # 网页抓取
│   ├── FileOperationTool.java          # 文件操作
│   ├── TerminalOperationTool.java      # 终端操作
│   ├── ResourceDownloadTool.java       # 资源下载
│   ├── PDFGenerationTool.java          # PDF 生成
│   └── TerminateTool.java              # 终止工具（标记任务完成）
├── rag/                                # 📚 RAG 检索增强模块
│   ├── LoveAppDocumentLoader.java      # 知识库文档加载器
│   ├── LoveAppVectorStoreConfig.java   # 向量存储配置
│   ├── LoveAppRagCloudAdvisorConfig.java  # 百炼云知识库
│   ├── LoveAppRagCustomAdvisorFactory.java # 自定义 RAG 顾问工厂
│   ├── LoveAppContextualQueryAugmenterFactory.java
│   ├── QueryRewriter.java              # Query 重写
│   ├── MyKeywordEnricher.java          # 关键词增强
│   ├── MyTokenTextSplitter.java        # 文档分词
│   └── PgVectorVectorStoreConfig.java  # PgVector 向量库配置
├── chatmemory/
│   └── FileBasedChatMemory.java        # 基于文件的对话记忆
├── advisor/
│   ├── MyLoggerAdvisor.java            # 日志顾问（调试用）
│   └── ReReadingAdvisor.java           # 推理增强顾问
├── config/
│   └── CorsConfig.java                 # 跨域配置
├── constant/
│   └── FileConstant.java              # 文件路径常量
└── demo/                               # 示例代码
    ├── invoke/                         # 多种 AI 调用方式示例
    │   ├── SdkAiInvoke.java            # SDK 方式调用
    │   ├── HttpAiInvoke.java           # HTTP 方式调用
    │   ├── SpringAiAiInvoke.java       # Spring AI 方式调用
    │   ├── LangChainAiInvoke.java      # LangChain 方式调用
    │   ├── OllamaAiInvoke.java         # Ollama 本地模型调用
    │   └── TestApiKey.java             # API Key 验证工具
    └── rag/
        └── MultiQueryExpanderDemo.java # 多查询扩展示例
```

### 3.2 AI 超级智能体（YuManus）工作原理

```
用户输入 → YuManus.runStream(message)
              │
              ▼
     ┌─ ReAct 循环 ───────────────┐
     │                             │
     │  1. think(): 调用大模型     │
     │     ├─ 分析用户需求          │
     │     ├─ 决定是否调用工具      │
     │     └─ 返回工具选择列表      │
     │           │                 │
     │  2. act(): 执行工具调用      │
     │     ├─ 调用选中的工具        │
     │     ├─ 收集执行结果          │
     │     └─ 结果加入消息上下文    │
     │           │                 │
     │  3. 判断是否调用 Terminate   │
│     ├─ 是 → 正常结束          │
     │     └─ 否 → 回到 think()    │
     │                             │
     └─────────────────────────────┘
              最多 20 步；达到上限或发生异常也会结束
```

**关键特点**：
- 模型生成工具调用请求，后端程序负责校验并执行相应工具
- 消息上下文维护：每次 think/act 的结果都保存到对话历史
- SSE 流式输出：每步结果实时推送到前端
- 禁用 Spring AI 内置工具执行：自己维护 `think → act` 循环以获得完全控制

### 3.3 AI 恋爱大师（LoveApp）工作原理

```
用户输入 → LoveApp.doChatByStream(message, chatId)
              │
              ▼
     ChatClient.prompt()
       .system(SYSTEM_PROMPT)     ← 设定情感专家人设
       .user(message)             ← 用户问题
       .advisors(
         MessageChatMemoryAdvisor ← 最多 20 条消息（按 chatId 隔离）
         MyLoggerAdvisor          ← 调试日志
       )
       .stream()                  ← SSE 流式输出
       .content()
```

**可扩展能力**（已有相关代码，但不等于 Controller 和前端已经接入）：
- **RAG 知识库问答**：加载 `resources/document/` 下的恋爱知识文档，向量检索增强回答
- **工具调用**：可接入搜索/文件等工具
- **MCP 服务调用**：通过 MCP 协议调用外部服务
- **结构化输出**：生成恋爱报告（标题 + 建议列表）
- **Query 重写**：优化用户查询，提高检索准确率

### 3.4 工具集一览

| 工具 | 类名 | 功能 |
|------|------|------|
| WebSearch | `WebSearchTool` | 网络搜索（需要 search-api.api-key） |
| WebScraping | `WebScrapingTool` | 网页内容抓取 |
| FileOperation | `FileOperationTool` | 文件读写操作 |
| TerminalOperation | `TerminalOperationTool` | 执行终端命令 |
| ResourceDownload | `ResourceDownloadTool` | 下载网络资源 |
| PDFGeneration | `PDFGenerationTool` | 生成 PDF 文件 |
| Terminate | `TerminateTool` | 模型主动声明任务完成的正常结束信号 |

### 3.5 API 接口

**基础路径**: `http://localhost:8123/api`

| 接口 | 方法 | 说明 | 参数 |
|------|------|------|------|
| `/ai/love_app/chat/sse` | GET | 恋爱大师 SSE 流式聊天 | `message`, `chatId` |
| `/ai/love_app/chat/sse_emitter` | GET | 同上（SseEmitter 实现） | `message`, `chatId` |
| `/ai/love_app/chat/server_sent_event` | GET | 同上（Flux 实现） | `message`, `chatId` |
| `/ai/love_app/chat/sync` | GET | 恋爱大师同步聊天 | `message`, `chatId` |
| `/ai/manus/chat` | GET | 超级智能体 SSE 流式聊天 | `message` |

Knife4j API 文档: `http://localhost:8123/api/swagger-ui.html`

---

## 四、前端详解

### 4.1 项目结构

```
yu-ai-agent-frontend/
├── index.html
├── package.json                # Vue 3 + Vue Router + Axios + SSE
├── vite.config.js              # Vite 配置 (端口 3000, @ 别名)
├── Dockerfile                   # Docker 部署配置
├── nginx.conf                   # Nginx 反向代理配置
└── src/
    ├── main.js                  # 入口：挂载 App + Router + Head
    ├── App.vue                  # 根组件：全局样式 + <router-view>
    ├── style.css                # 基础样式
    ├── router/index.js          # 路由配置（3 个页面 + 导航守卫）
    ├── api/index.js             # API 层：Axios + SSE 封装
    ├── views/
    │   ├── Home.vue             # 首页（赛博朋克风格入口）
    │   ├── LoveMaster.vue       # AI 恋爱大师聊天页
    │   └── SuperAgent.vue       # AI 超级智能体聊天页
    └── components/
        ├── ChatRoom.vue         # 通用聊天组件（消息列表 + 输入框）
        ├── AppFooter.vue        # 页脚组件
        ├── AiAvatarFallback.vue # AI 头像回退组件
        └── HelloWorld.vue       # 脚手架残留（未使用）
```

### 4.2 路由与页面

| 路径 | 页面 | 说明 |
|------|------|------|
| `/` | Home | 首页，展示两个应用入口卡片 |
| `/love-master` | LoveMaster | 恋爱大师聊天，粉色主题 |
| `/super-agent` | SuperAgent | 超级智能体聊天，蓝色主题 |

### 4.3 数据流

```
用户输入消息
    │
    ▼
ChatRoom.vue  ──emit('send-message', message)──→ LoveMaster.vue / SuperAgent.vue
                                                        │
                                                   调用 API 层
                                                        │
                                              chatWithLoveApp() /
                                              chatWithManus()
                                                        │
                                                   创建 EventSource
                                                   (SSE 长连接)
                                                        │
                                              onmessage 回调
                                              ├─ 收到数据 → 更新 messages
                                              └─ [DONE]  → 关闭连接
```

### 4.4 两大应用的差异

| 维度 | AI 恋爱大师 | AI 超级智能体 |
|------|------------|-------------|
| 主题色 | 粉色 (#ff6b8b) | 蓝色 (#3f51b5) |
| 会话管理 | 有 chatId，支持多轮对话记忆 | 无 chatId，每次独立 |
| 消息展示 | 逐步拼接到同一条消息 | 按句子分割成多个气泡 |
| 欢迎语 | "欢迎来到AI恋爱大师..." | "你好，我是AI超级智能体..." |
| 后端接口 | `/love_app/chat/sse` | `/manus/chat` |

---

### 4.5 功能启用状态

“代码中存在”不代表“网页已经可以使用”。判断一个功能是否完整可用，需要同时检查后端方法、Controller 接口、前端接入和外部配置。

| 功能 | 核心代码 | Controller 接口 | 当前前端接入 | 额外条件 |
|------|:--------:|:---------------:|:------------:|----------|
| 恋爱大师基础聊天 | ✅ | ✅ | ✅ | DashScope API Key |
| YuManus 智能体 | ✅ | ✅ | ✅ | DashScope API Key；搜索功能还需 Search API Key |
| 本地 RAG | ✅ | ❌（主要通过测试调用） | ❌ | Embedding 模型和 `resources/document/` 文档 |
| PgVector RAG | ✅ 可选配置 | ❌ | ❌ | PostgreSQL + pgvector，并恢复数据源自动配置 |
| LoveApp 工具调用 | ✅ | ❌（主要通过测试调用） | ❌ | 对应工具凭证及安全限制 |
| LoveApp MCP 调用 | ✅ | ❌（主要通过测试调用） | ❌ | 启用 MCP 客户端并启动 MCP 服务端 |
| 结构化恋爱报告 | ✅ | ❌（主要通过测试调用） | ❌ | DashScope API Key |

---

## 五、配置说明

### application.yml 关键配置

```yaml
spring:
  ai:
    dashscope:
      api-key: ${SPRING_AI_DASHSCOPE_API_KEY} # 必填：通过环境变量注入，不要提交真实密钥
      chat.options.model: qwen-plus   # 使用的模型
    ollama:
      base-url: http://localhost:11434  # 本地 Ollama（备用）

server:
  port: 8123                           # 后端端口
  servlet.context-path: /api           # 接口前缀

search-api:
  api-key: ${SEARCH_API_API_KEY}       # 网络搜索工具需要，同样不要提交真实密钥
```

> 安全要求：如果真实密钥曾写入源码或提交到 Git，应立即在服务商控制台吊销并重新生成。仅删除当前文件中的密钥，不能消除 Git 历史中的泄露记录。

### 关键依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| Spring Boot | 3.4.4 | 基础框架 |
| spring-ai-alibaba | 1.0.0.2 | 阿里百炼 DashScope 集成 |
| spring-ai | 1.0.0 | Spring AI 框架 |
| Hutool | - | Java 工具库 |
| Knife4j | - | API 文档 |
| Lombok | - | 代码简化 |

---

## 六、启动指南

### 前置条件

- Java 21+
- Node.js 18 LTS 或更高版本（项目未声明 `engines.node`，启动前请用 `node -v` 确认）
- Maven（或用项目自带的 `mvnw`）

### 1. 后端启动

```bash
cd yu-ai-agent-master

# 推荐在当前终端配置环境变量，不要把真实密钥写入或提交到 application.yml

# Windows PowerShell
$env:SPRING_AI_DASHSCOPE_API_KEY="你的百炼 API Key"
$env:SEARCH_API_API_KEY="你的搜索 API Key"

# Windows
mvnw.cmd spring-boot:run

# Linux / Mac
./mvnw spring-boot:run
```

启动后访问: `http://localhost:8123/api/swagger-ui.html`

### 2. 前端启动

```bash
cd yu-ai-agent-frontend

npm install
npm run dev
```

启动后访问: `http://localhost:3000`

---

## 七、当前状态与扩展空间

### 已实现且可直接使用
- ✅ AI 恋爱大师基础对话（多轮记忆 + SSE 流式）
- ✅ AI 超级智能体 ReAct 工具调用（SSE 流式）
- ✅ 前端完整 UI（首页 + 两个聊天页 + 响应式布局）

### 已实现但尚未形成完整前后端链路
- 🔧 本地 RAG：默认使用内存型 `SimpleVectorStore`，不要求 PostgreSQL，但当前页面没有接入 RAG 接口
- 🔧 MCP 服务调用：主应用配置默认注释，需要启用客户端、构建并启动 MCP 服务端
- 🔧 PgVector 向量存储（需要 PostgreSQL + pgvector 插件）
- 🔧 网络搜索工具（需要搜索 API Key）
- 🔧 Ollama 本地模型（需要安装 Ollama 并拉取模型）

### 代码中注释掉的功能
- `FileBasedChatMemory`：文件持久化对话记忆（当前使用内存记忆）
- `ReReadingAdvisor`：推理增强顾问
- `LoveAppRagCustomAdvisorFactory`：自定义 RAG 过滤（按恋爱状态过滤文档）

---

## 八、安全边界

本项目包含终端、文件、下载和网页抓取等高风险工具，适合本地教学和受控实验。公开部署前至少需要：

- 用户登录、鉴权、限流和调用额度控制
- 终端命令白名单、超时、资源限制和隔离沙箱
- 文件操作限定在独立工作目录，并阻止路径穿越
- 下载地址和网页抓取目标校验，防止 SSRF 和恶意文件
- 工具调用审计日志；高风险操作增加人工确认
- API Key 通过环境变量或密钥管理服务注入

未经这些限制，不应将 `TerminalOperationTool`、`FileOperationTool`、`ResourceDownloadTool` 等能力直接暴露到公网。

---

## 九、常见启动问题

| 现象 | 优先检查 |
|------|----------|
| 后端无法编译 | `java -version` 是否为 Java 21；Maven 依赖是否下载成功 |
| 模型返回鉴权错误 | `SPRING_AI_DASHSCOPE_API_KEY` 是否存在、有效且当前终端可见 |
| 搜索工具失败 | `SEARCH_API_API_KEY` 是否配置；第三方搜索服务是否可访问 |
| 前端打不开 | `npm install` 是否成功；3000 端口是否被占用 |
| 页面有消息但没有流式效果 | 浏览器 EventSource、后端 SSE 日志、代理缓冲和超时设置 |
| RAG 没有生效 | 当前调用的是否真的是 RAG 方法；知识文档是否加载成功 |
| PgVector 启动失败 | PostgreSQL、pgvector、数据源配置以及 `DataSourceAutoConfiguration` 排除项 |
| MCP 无法连接 | MCP 子项目是否已构建和启动；客户端配置是否取消注释；传输方式是否一致 |

---

## 十、文档使用建议

- 本文负责说明“项目是什么、模块如何协作、哪些功能已经接通”。
- 具体学习顺序、逐段代码解释和练习任务请参考 `LEARNING_GUIDE.md`。
- 每次修改接口、配置或功能接入状态后，应同步更新本文的“功能启用状态”。
