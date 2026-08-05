# 鱼皮 AI 超级智能体 — 新手学习指南

## 阅读指南

- 📖 本文档按**从零到懂**的路线组织，共 5 个阶段
- ⏱️ 每个阶段标注了预计学习时间，总计约 **12-16 小时**
- 💡 "为什么要这样做"是本文档的核心 — 不只讲代码，更讲**设计意图**
- 🔗 文中文件路径均可点击跳转

---

## 第一阶段：先把项目跑起来（约 30 分钟）

> **目标**：建立感性认识。先看到效果，后面学代码才能对号入座。

### 1.1 启动后端

```bash
cd yu-ai-agent-master

# Windows
mvnw.cmd spring-boot:run

# Mac/Linux
./mvnw spring-boot:run
```

启动成功后访问：
- Swagger 接口文档：`http://localhost:8123/api/swagger-ui.html`
- 健康检查：`http://localhost:8123/api/health`

### 1.2 启动前端

打开第二个终端：

```bash
cd yu-ai-agent-frontend
npm install      # 仅首次
npm run dev
```

访问 `http://localhost:3000`

### 1.3 动手试试

1. 首页点 **"AI恋爱大师"** → 输入 "我喜欢一个女生，但不知道怎么开口"，看回复
2. 点返回，进入 **"AI超级智能体"** → 输入 "帮我搜索一下今天的热点新闻"，看 Agent 调用搜索工具
3. 观察两个聊天页的差异：恋爱大师的消息是**逐步拼接到一条**，超级智能体是**分成多个气泡**

### 💡 先思考几个问题（带着问题往下学）

- 为什么恋爱大师回复特别快，超级智能体回复要等一会儿？
- 为什么同样的聊天界面出现在两个页面，但长得不一样？
- 前端怎么知道 AI 还在"打字中"？
- 后端怎么把一句话逐字推送给前端的？（不是一次性返回）

---

## 第二阶段：前端 — 数据是怎么流动的（约 2-3 小时）

> **目标**：理解从"用户点击发送"到"屏幕上出现 AI 回复"的完整前端链路。

### 2.1 路由系统：URL 怎么对应到页面

**文件**：[src/router/index.js](yu-ai-agent-frontend/src/router/index.js)

```
用户访问 /            → 显示 Home.vue（首页）
用户访问 /love-master  → 显示 LoveMaster.vue（恋爱大师）
用户访问 /super-agent  → 显示 SuperAgent.vue（超级智能体）
```

**关键代码**：
```js
const routes = [
  { path: '/', name: 'Home', component: () => import('../views/Home.vue') },
  { path: '/love-master', name: 'LoveMaster', component: () => import('../views/LoveMaster.vue') },
  { path: '/super-agent', name: 'SuperAgent', component: () => import('../views/SuperAgent.vue') }
]
```

**为什么要这样做**：
- 三个路径、三个页面，这是 SPA（单页应用）的基本结构
- `() => import(...)` 是**懒加载**：用户不访问的页面不会下载，首页加载更快
- 导航守卫 `router.beforeEach` 负责自动改浏览器标题栏文字

**🤔 新手理解**：路由就像一个"遥控器"——用户点不同按钮（路径），屏幕切换到不同频道（页面组件），但整个 APP 没有刷新。

---

### 2.2 API 层：前端怎么跟后端"说话"

**文件**：[src/api/index.js](yu-ai-agent-frontend/src/api/index.js)

这个文件是整个项目的**数据通道**，做了两件事：

#### ① Axios 实例（普通 HTTP 请求用）

```js
const request = axios.create({
  baseURL: 'http://localhost:8123/api',  // 所有请求自动加这个前缀
  timeout: 60000                          // 60 秒超时
})
```

**为什么要这样做**：抽出一个公共实例，就不用每次请求都写完整的 `http://localhost:8123/api/xxx`，改地址也只需要改一处。

#### ② SSE 连接（AI 流式对话用）

```js
export const connectSSE = (url, params, onMessage, onError) => {
  const fullUrl = `${API_BASE_URL}${url}?${queryString}`
  const eventSource = new EventSource(fullUrl)

  eventSource.onmessage = event => {
    // 收到数据就回调
    if (data === '[DONE]') {
      onMessage('[DONE]')   // 标记结束
    } else {
      onMessage(data)       // 正常数据
    }
  }
  return eventSource  // 返回实例，方便后续关闭连接
}
```

### 🌟 重点概念：SSE vs 普通 HTTP vs WebSocket

| | 普通 HTTP | SSE | WebSocket |
|------|----------|-----|-----------|
| 方向 | 客户端请求 → 服务端一次性返回 | 服务端**持续推送**给客户端 | 双向通信 |
| 连接 | 请求-即断开 | 长连接，服务端不挂断 | 长连接，双向 |
| 场景 | 查询订单、加载列表 | **AI 对话流式输出**、股票行情 | 在线聊天、游戏 |
| 复杂度 | 最简单 | 简单（浏览器原生 `EventSource`） | 需要额外库 |

**为什么这个项目用 SSE 而不是 WebSocket**：
1. AI 对话只需要服务端推送 → 客户端，不需要客户端主动推（SSE 够用）
2. SSE 是 HTTP 协议，天然走 80/443 端口，不会被防火墙拦截
3. 浏览器自带 `EventSource` API，无需引入第三方库
4. 断线自动重连（浏览器原生支持）

**🤔 新手理解**：把 SSE 想象成"直播"——你打开直播间（建立连接），主播一直说（服务端推送），你不会发弹幕问下一句是什么。把 WebSocket 想象成"视频通话"——双方都可以随时说话。

---

### 2.3 ChatRoom 组件：聊天界面的"万能模板"

**文件**：[src/components/ChatRoom.vue](yu-ai-agent-frontend/src/components/ChatRoom.vue)

这是整个前端**最重要的组件**，恋爱大师和超级智能体都在复用它。

#### 组件通信：props 向下，emit 向上

```
LoveMaster.vue（父组件）
    │
    │  props: messages, connectionStatus, aiType    ← 父传给子
    │  emit:  send-message                          ← 子通知父
    │
    ▼
ChatRoom.vue（子组件，纯展示+输入）
```

**为什么要这样设计**：
- ChatRoom 只负责展示消息和收集输入，不关心"这条消息怎么发给后端"
- 恋爱大师和超级智能体**发送消息的方式不同**（一个传 chatId，一个不传），所以"发送"这个动作留给父组件处理
- 这叫 **"容器组件 vs 展示组件"** 模式——降低耦合，改一个不影响另一个

#### 关键代码解读

```js
// 1. 定义接收的数据
const props = defineProps({
  messages: Array,           // 消息列表（由父组件维护）
  connectionStatus: String,  // 连接状态（connecting/disconnected/error）
  aiType: String             // 'love' 或 'super'，决定左边头像样式
})

// 2. 定义抛出的事件
const emit = defineEmits(['send-message'])

// 3. 发送消息：不自己发请求，而是通知父组件
const sendMessage = () => {
  emit('send-message', inputMessage.value)
  inputMessage.value = ''
}

// 4. 自动滚动：监听 messages 变化，有新消息就滚到底部
watch(() => props.messages.length, () => { scrollToBottom() })
watch(() => props.messages.map(m => m.content).join(''), () => { scrollToBottom() })
```

**为什么要监听两次**：
- 第一次 `messages.length`：新消息出现时滚动
- 第二次 `messages.map(m => m.content).join('')`：**同一条消息内容变化时**也滚动（恋爱大师就是逐步拼接到同一条消息，length 不变但 content 在变）

---

### 2.4 恋爱大师页面：消息"生长"模式

**文件**：[src/views/LoveMaster.vue](yu-ai-agent-frontend/src/views/LoveMaster.vue)

#### 核心逻辑

```js
const sendMessage = (message) => {
  addMessage(message, true)          // 1. 先添加用户消息
  const aiMessageIndex = messages.value.length
  addMessage('', false)              // 2. 创建一条空的 AI 消息（占位）
  eventSource = chatWithLoveApp(message, chatId.value)  // 3. 建立 SSE

  eventSource.onmessage = (event) => {
    // 4. 逐步拼接内容到同一条 AI 消息
    messages.value[aiMessageIndex].content += event.data
  }
}
```

**为什么要逐步拼接而不是新建消息**：
- 模拟人类打字的效果，用户感觉更自然
- 用户体验：看起来 AI 在"边想边说"
- 技术上：只需要更新数组里一个元素的 content，DOM diff 很小，性能好

#### chatId 是什么

```js
const generateChatId = () => 'love_' + Math.random().toString(36).substring(2, 10)
```

**为什么要 chatId**：后端用 chatId 来区分不同会话，实现**多轮对话记忆**。同一个 chatId 下的所有消息共享上下文，AI 能记住你之前说过什么。

---

### 2.5 超级智能体页面：消息"气泡"模式

**文件**：[src/views/SuperAgent.vue](yu-ai-agent-frontend/src/views/SuperAgent.vue)

与恋爱大师的核心区别：

| | 恋爱大师 | 超级智能体 |
|------|---------|----------|
| 消息策略 | 逐步拼接同一条消息 | **按句子分割成多个气泡** |
| 有无 chatId | ✅ 有（保持多轮记忆） | ❌ 无（每次独立对话） |
| 气泡类型 | 就一种 | `ai-answer`（回答）、`ai-final`（最终）、`ai-error`（出错） |

```js
// 核心判断：检测到句号、问号等 → 创建新气泡
const chineseEndPunctuation = ['。', '！', '？', '…']
const lastChar = data.charAt(data.length - 1)
if (chineseEndPunctuation.includes(lastChar) || combinedText.length > 40) {
  createBubble(combinedText)  // 切一刀，生成一个新气泡
}

// 气泡之间有至少 800ms 的间隔，模拟思考过程
const minBubbleInterval = 800
```

**为什么要分成气泡**：
- YuManus 是 ReAct Agent，每步 think → act 返回一条结果，多个结果之间是**不同的步骤**
- 分成气泡让用户看到 Agent 的**思考过程**——"第一步搜索了什么"、"第二步分析出了什么"
- 比恋爱大师的消息更有"结构感"

---

## 第三阶段：后端核心 — AI 到底怎么"思考"（约 4-6 小时）

> **目标**：理解后端两大核心应用的工作方式，以及 Agent 的基础概念。

### 🌟 前置知识：什么是 Agent（智能体）

**不是魔法，而是一个循环**：

```
人类：帮我搜索今天天气，然后总结一下穿什么衣服

Agent 循环：
  ┌─────────────────────────────────┐
  │ Step 1: think()                 │
  │   AI: "我需要先搜索天气"        │
  │   → 决定调用 WebSearchTool      │
  │                                 │
  │ Step 1: act()                   │
  │   → 执行搜索，得到 "28°C 晴天"  │
  │                                 │
  │ Step 2: think()                 │
  │   AI: "天气是晴天28度，建议..."  │
  │   → 不调用工具，直接回答         │
  │   → 调用 TerminateTool 结束     │
  └─────────────────────────────────┘
```

用代码表示就是：
```java
while (state == RUNNING && 没超过最大步数) {
    think();   // 让 AI 想下一步做什么
    act();     // 执行 AI 决定做的动作
}
```

**为什么要有 Agent**：单次问答只能回答"知道"的事，Agent 能**调用工具**获取新信息，再基于新信息回答问题——这是从"知识库"到"行动力"的跨越。

---

### 3.1 继承链：四层抽象的设计意图

```
BaseAgent                   ← 管理状态、循环、SSE
  └─ ReActAgent             ← 定义 think → act 模式
       └─ ToolCallAgent     ← 实现 think/act 的工具调用逻辑
            └─ YuManus      ← 设置系统提示词和参数
```

**为什么要分四层，而不是写一个类**：

| 层级 | 职责 | 如果合并到一起会怎样 |
|------|------|---------------------|
| BaseAgent | 循环控制、状态管理、消息上下文、SSE | 每写一个 Agent 都要重写循环逻辑 |
| ReActAgent | 定义 think/act 接口 | 换不同的 Agent 模式（如 Plan-Execute）要改所有代码 |
| ToolCallAgent | 工具调用的具体实现 | 不用工具的 Agent 也要带着工具代码 |
| YuManus | 特定的人格、提示词、参数 | 一个类里混着通用逻辑和业务逻辑 |

**这就是"面向对象"的核心价值：把变和不变分开**。

**🤔 新手理解**：就像乐高——BaseAgent 是底板，ReActAgent 加了电机，ToolCallAgent 装了手臂，YuManus 是最后的成品机器人。每一层只管自己的事。

---

### 3.2 BaseAgent：一切的基础

**文件**：[src/main/java/com/yupi/yuaiagent/agent/BaseAgent.java](src/main/java/com/yupi/yuaiagent/agent/BaseAgent.java)

```java
public abstract class BaseAgent {
    private String name;                    // 智能体名称
    private String systemPrompt;            // 系统提示词（人设）
    private String nextStepPrompt;          // 每步指导提示词
    private AgentState state = IDLE;        // 当前状态
    private int currentStep = 0;            // 当前步数
    private int maxSteps = 10;              // 最大步数（防止死循环）
    private ChatClient chatClient;          // 大模型客户端
    private List<Message> messageList;      // 消息上下文（记忆）
}
```

**为什么要有 maxSteps**：大模型可能陷入"调用工具 → 不满意 → 再调用 → 还不满意"的死循环，maxSteps 是硬兜底，10 步后强制终止。

**为什么 messageList 是 List\<Message\>**：Agent 的每一步对话都要记住——用户说了什么、AI 回了什么、工具返回了什么——全部保存在这个列表里，下次 think() 时带着整个历史去问大模型。

#### run() vs runStream() — 两种执行模式

```java
// 普通模式：等全部完成再返回
public String run(String userPrompt) {
    for (循环) {
        stepResult = step();
        results.add(stepResult);
    }
    return String.join("\n", results);  // 所有步骤拼一起返回
}

// 流式模式：每完成一步就推送
public SseEmitter runStream(String userPrompt) {
    CompletableFuture.runAsync(() -> {    // 异步执行，不阻塞主线程
        for (循环) {
            stepResult = step();
            sseEmitter.send(stepResult);  // 立即推送给前端
        }
        sseEmitter.complete();            // 推送完成标记
    });
    return sseEmitter;  // 立即返回发射器
}
```

**为什么要用 CompletableFuture.runAsync**：Spring MVC 的请求处理线程不能长时间阻塞。`runAsync` 把耗时工作交给线程池，主线程立刻返回 SseEmitter，SSE 连接才能建立。

---

### 3.3 ReActAgent：模式的骨架

**文件**：[src/main/java/com/yupi/yuaiagent/agent/ReActAgent.java](src/main/java/com/yupi/yuaiagent/agent/ReActAgent.java)

这个类**只有 30 行代码**，但它是整个 Agent 架构的核心：

```java
public abstract class ReActAgent extends BaseAgent {
    public abstract boolean think();    // 子类必须实现：怎么思考
    public abstract String act();       // 子类必须实现：怎么行动

    @Override
    public String step() {              // 单步 = think + act
        boolean shouldAct = think();    // 1. 思考：要不要行动
        if (!shouldAct) {
            return "思考完成 - 无需行动";
        }
        return act();                   // 2. 行动：执行工具
    }
}
```

**ReAct = Reasoning + Acting（推理 + 行动）**，这是 2023 年 Google 提出的一种让大模型更可靠地使用工具的 Prompt 工程方法。

**为什么 step() 不包含 plan（规划）和 reflect（反思）**：这是教学项目的简化实现。完整的 Agent 模式应该是：

```
Plan → Think → Act → Observe → Reflect → (循环或结束)
```

项目里把 Plan 简化成了系统提示词，把 Reflect 简化成了"AI 自己判断要不要继续"。

---

### 3.4 ToolCallAgent：工具调用的发动机

**文件**：[src/main/java/com/yupi/yuaiagent/agent/ToolCallAgent.java](src/main/java/com/yupi/yuaiagent/agent/ToolCallAgent.java)

这是整个项目**最复杂的类**。核心是 think() 和 act() 两个方法。

#### think() — "我该用什么工具？"

```java
public boolean think() {
    // 1. 添加"下一步做什么"的提示词
    messageList.add(new UserMessage(getNextStepPrompt()));

    // 2. 调用大模型，传上所有历史消息 + 所有可用工具
    ChatResponse chatResponse = getChatClient()
        .prompt(new Prompt(messageList))
        .system(getSystemPrompt())     // "你是 YuManus，一个全能助手..."
        .tools(availableTools)         // WebSearch、FileOp、Terminate...
        .call()
        .chatResponse();

    // 3. 解析大模型的回复：要调用工具，还是直接回答？
    List<ToolCall> toolCallList = chatResponse.getResult().getOutput().getToolCalls();

    if (toolCallList.isEmpty()) {
        return false;    // AI 觉得不需要工具，直接回答
    } else {
        return true;     // AI 想调用工具，进入 act()
    }
}
```

**为什么要加 nextStepPrompt**：大模型是"无状态"的——每次都从头看上下文。没有这个提示词，AI 可能忘了"我现在在 Agent 循环中，需要主动选工具"。这就像每次下棋前提醒一句"该你下了"。

**为什么要禁用内部工具执行**：
```java
this.chatOptions = DashScopeChatOptions.builder()
    .withInternalToolExecutionEnabled(false)   // 禁用！
    .build();
```

Spring AI 默认是"自动帮你执行工具调用"的，但这里要**手动控制 think → act 循环**。因为：
- 需要在每步之间插入自己的逻辑（状态管理、日志、计数）
- 需要控制流程——什么时候终止循环由 TerminateTool 决定，不是 AI 自己说了算

#### act() — "执行它！"

```java
public String act() {
    // 调用 Spring AI 的工具执行管理器
    ToolExecutionResult result = toolCallingManager.executeToolCalls(prompt, toolCallChatResponse);

    // 更新消息上下文（工具调用的结果加入历史）
    setMessageList(result.conversationHistory());

    // 如果调用了 TerminateTool → 标记任务完成
    if (调用了doTerminate) {
        setState(AgentState.FINISHED);   // 下次循环检查到这里就停了
    }
    return 工具执行结果;
}
```

**为什么 TerminateTool 这么重要**：它是让 Agent"停下来"的**唯一开关**。没有它，Agent 会一直 think → act → think → act 直到 maxSteps 耗尽。

---

### 3.5 YuManus：最后一步——配置人设

**文件**：[src/main/java/com/yupi/yuaiagent/agent/YuManus.java](src/main/java/com/yupi/yuaiagent/agent/YuManus.java)

```java
public class YuManus extends ToolCallAgent {
    public YuManus(ToolCallback[] allTools, ChatModel dashscopeChatModel) {
        super(allTools);

        this.setSystemPrompt("""
            You are YuManus, an all-capable AI assistant,
            aimed at solving any task presented by the user.
            """);

        this.setNextStepPrompt("""
            Based on user needs, proactively select the most
            appropriate tool. For complex tasks, break down
            the problem step by step.
            """);

        this.setMaxSteps(20);  // 比默认的 10 步翻倍
    }
}
```

**为什么只有 30 行**：前面三层已经把所有通用逻辑抽走了。YuManus 只需要做三件事：你是谁（systemPrompt）、每步怎么决策（nextStepPrompt）、最多几步（maxSteps）。**好的抽象让最终实现类非常薄**。

---

### 3.6 LoveApp：另一种路径 —— 不用 Agent 的简单对话

**文件**：[src/main/java/com/yupi/yuaiagent/app/LoveApp.java](src/main/java/com/yupi/yuaiagent/app/LoveApp.java)

LoveApp 的架构比 YuManus 简单得多：

```java
// 创建一个"已经配好的"ChatClient
ChatClient chatClient = ChatClient.builder(dashscopeChatModel)
    .defaultSystem("扮演深耕恋爱心理领域的专家...")     // 人设写死
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory)    // 自动记住最近 20 轮对话
            .build(),
        new MyLoggerAdvisor()                           // 打日志
    )
    .build();

// 对话就是一行代码
public Flux<String> doChatByStream(String message, String chatId) {
    return chatClient.prompt()
        .user(message)
        .advisors(spec -> spec.param("conversationId", chatId))  // 按 chatId 区分会话
        .stream()                                    // 流式输出
        .content();                                  // 只要文本部分
}
```

**为什么 LoveApp 不用 Agent 循环**：
- 恋人咨询是"聊天"不是"任务"，不需要调用工具搜索或生成 PDF
- ChatClient 的 advisor 机制（MessageChatMemoryAdvisor）已经自动处理了多轮记忆，不需要手动维护 messageList
- 更简单、更快、更便宜——一次大模型调用就够了，不像 Agent 可能调 5-10 次

---

### 3.7 两种模式的对比总结

| | LoveApp（简单对话） | YuManus（Agent） |
|------|---------|------|
| 模式 | ChatClient + Advisor | BaseAgent → ReAct → ToolCall |
| 适用场景 | 聊天、咨询、创作 | 需要工具辅助的复杂任务 |
| 每次对话调用大模型次数 | 1 次 | 2-20 次 |
| 对话记忆 | Advisor 自动管理 | 手动维护 messageList |
| 代码复杂度 | 简单，核心 20 行 | 复杂，四层继承 + 工具系统 |

---

## 第四阶段：扩展能力（约 3-4 小时）

> **目标**：理解 Spring AI 提供的进阶能力。

### 4.1 RAG — 让 AI 能"翻书"回答问题

#### 是什么

RAG（Retrieval-Augmented Generation，检索增强生成）解决一个核心问题：**大模型不知道你私有文档里的内容**。

```
没有 RAG：
  用户："我们公司的年假政策是什么？"
  AI："抱歉，我不知道你们公司的具体政策..."

有 RAG：
  用户："我们公司的年假政策是什么？"
  → 系统先去知识库搜索"年假政策"
  → 找到相关文档内容
  → 把文档内容塞给 AI："根据以下信息回答：{文档内容}\n\n用户问题：我们公司的年假政策是什么？"
  → AI："根据公司规定，入职满1年享有5天年假..."
```

#### 项目里怎么实现的

```java
// LoveApp.java 中的 RAG 对话方法
public String doChatWithRag(String message, String chatId) {
    // 1. Query 重写：把口语化的问法改成更适合检索的表达
    String rewrittenMessage = queryRewriter.doQueryRewrite(message);

    ChatResponse chatResponse = chatClient.prompt()
        .user(rewrittenMessage)
        // 2. 关键：加上 QuestionAnswerAdvisor，自动做向量检索
        .advisors(new QuestionAnswerAdvisor(loveAppVectorStore))
        .call()
        .chatResponse();
}
```

**为什么要 Query 重写**：
- 用户："女生生气了怎么哄" → 口语化
- 重写后："恋爱中 女生生气 如何安抚 方法" → 更适合搜索
- QueryTransformer 会调用大模型做这件事

**为什么文档要切成小块**：向量检索是语义匹配，小块文字更容易找到相关段落。就像书签夹到具体的段落，而不是夹到整个章节。

---

### 4.2 工具系统 — AI 的"手"

#### 工具是怎么注册的

**文件**：[src/main/java/com/yupi/yuaiagent/tools/ToolRegistration.java](src/main/java/com/yupi/yuaiagent/tools/ToolRegistration.java)

```java
@Bean
public ToolCallback[] allTools() {
    return ToolCallbacks.from(
        new FileOperationTool(),       // 文件操作
        new WebSearchTool(apiKey),     // 网络搜索
        new WebScrapingTool(),         // 网页抓取
        new ResourceDownloadTool(),    // 资源下载
        new TerminalOperationTool(),   // 终端命令
        new PDFGenerationTool(),       // PDF 生成
        new TerminateTool()            // 终止标记
    );
}
```

#### 工具怎么被 AI "看到"

Spring AI 通过 `@Tool` 注解自动生成工具描述：

```java
// WebSearchTool.java
@Tool(description = "Search for information from Baidu Search Engine")
public String searchWeb(
    @ToolParam(description = "Search query keyword") String query
) {
    // ...
}
```

这段注解会被 Spring AI 自动转成类似这样的格式，塞进发给大模型的 Prompt 里：

```
可用工具：
- searchWeb: Search for information from Baidu Search Engine
  参数 query: Search query keyword
```

大模型看到这个，就知道有搜索工具可用，并根据用户的需求自主决定是否调用。

#### AI 怎么"决定"用哪个工具

这是大模型内置的能力（Function Calling）——不是代码逻辑判断的，是**大模型自己推理**的：

> 用户："帮我搜一下今天天气"
> 大模型推理："用户要天气 → 我需要搜索 → 调用 searchWeb(query='今天天气')"
> 
> 用户："帮我写个 Hello World"
> 大模型推理："这不需要搜索 → 不调用任何工具，直接用我的知识回答"

**为什么 TerminateTool 的 doTerminate 不需要参数也不需要返回有意义的结果**：它只是一个**信号**——"任务结束"。AI 调用它就等于说"我做完了"，代码检测到这个调用就把 state 改为 FINISHED。

---

### 4.3 Spring AI 框架——到底帮我们做了什么

如果没有 Spring AI，你要自己：

1. **HTTP 调用大模型** → 写 OkHttp/WebClient，处理超时、重试、流式解析
2. **拼接 Prompt** → 手动拼系统提示词 + 工具描述 + 对话历史
3. **解析工具调用** → 自己解析 JSON，识别哪个工具、什么参数
4. **向量检索** → 自己调 Embedding API、做余弦相似度计算
5. **SSE 实现** → 自己管理线程、处理断连重连

Spring AI 把这些都封装了。**项目里最有价值的部分不是调用 Spring AI，而是在 Spring AI 之上搭建的 Agent 架构层**——这个是框架没有提供的。

---

## 第五阶段：串联总结（约 1-2 小时）

> **目标**：理解整体架构设计的意图。

### 5.1 分层架构的意图

```
┌─────────────────────────────────────────┐
│  前端展示层   │  Vue 3 + Router + SSE    │  ← 怎么给用户看
├─────────────────────────────────────────┤
│  API 接入层   │  AiController            │  ← 怎么暴露接口
├─────────────────────────────────────────┤
│  应用编排层   │  LoveApp / YuManus       │  ← 怎么组织 AI 能力
├─────────────────────────────────────────┤
│  Agent 框架层 │  BaseAgent → ToolCall    │  ← 通用的智能体模式
├─────────────────────────────────────────┤
│  AI 基础设施  │  Spring AI + DashScope    │  ← 底层框架 + 模型
└─────────────────────────────────────────┘
```

**每层只依赖下面一层**，改工具不影响到 Agent 框架，改前端不影响后端逻辑。

### 5.2 设计决策一览

| 决策 | 选择 | 为什么 |
|------|------|--------|
| 前端通信方式 | SSE | 单向推送就够了，比 WebSocket 简单可靠 |
| 后端框架 | Spring Boot + Spring AI | Java 生态最佳选择，DI 原生支持 |
| 大模型 | 通义千问 qwen-plus | 国内访问快，百炼平台成熟 |
| Agent 模式 | ReAct | 经典论文模式，适合教学演示 |
| 对话记忆 | 内存（当前）/ 文件（可选） | 简单优先，可扩展 |
| 工具注册 | Spring Bean 集合 | 自然融入 Spring DI 体系 |
| 前端组件复用 | props/emit 模式 | Vue 3 标准范式，无额外依赖 |

### 5.3 学习延伸

如果把这个项目吃透了，下一步可以往几个方向深入：

1. **LangChain / LangGraph**：Python 生态的 Agent 框架，支持更复杂的多 Agent 协作
2. **MCP 协议**：让 Agent 能调用任何遵循 MCP 的外部服务
3. **Prompt Engineering**：系统提示词的设计是 Agent 表现好坏的关键
4. **向量数据库**：Milvus / Pinecone / PgVector，RAG 的底层存储
5. **Agent 评测**：如何评估 Agent 的回答质量？准确率、召回率、工具选择正确率

---

## 附录：关键文件速查表

### 后端（按学习顺序）

| 文件 | 核心内容 | 学习重点 |
|------|----------|----------|
| [application.yml](src/main/resources/application.yml) | 配置 | 端口、模型选择、API Key |
| [AiController.java](src/main/java/com/yupi/yuaiagent/controller/AiController.java) | 5 个 API 端点 | SSE 三种实现方式 |
| [LoveApp.java](src/main/java/com/yupi/yuaiagent/app/LoveApp.java) | 简单对话模式 | ChatClient 用法、Advisor 机制 |
| [BaseAgent.java](src/main/java/com/yupi/yuaiagent/agent/BaseAgent.java) | Agent 基类 | 循环控制、SSE 流式、状态管理 |
| [ReActAgent.java](src/main/java/com/yupi/yuaiagent/agent/ReActAgent.java) | ReAct 模式骨架 | think/act 接口定义 |
| [ToolCallAgent.java](src/main/java/com/yupi/yuaiagent/agent/ToolCallAgent.java) | 工具调用实现 | think() 调用大模型、act() 执行工具 |
| [YuManus.java](src/main/java/com/yupi/yuaiagent/agent/YuManus.java) | 超级智能体 | 提示词设计、参数配置 |
| [ToolRegistration.java](src/main/java/com/yupi/yuaiagent/tools/ToolRegistration.java) | 工具注册 | 7 个工具的组织方式 |
| [WebSearchTool.java](src/main/java/com/yupi/yuaiagent/tools/WebSearchTool.java) | 搜索工具 | @Tool 注解用法 |
| [TerminateTool.java](src/main/java/com/yupi/yuaiagent/tools/TerminateTool.java) | 终止工具 | 控制 Agent 循环结束 |
| [LoveAppDocumentLoader.java](src/main/java/com/yupi/yuaiagent/rag/LoveAppDocumentLoader.java) | 文档加载 | Markdown → 向量化 |
| [QueryRewriter.java](src/main/java/com/yupi/yuaiagent/rag/QueryRewriter.java) | Query 重写 | 用大模型优化搜索查询 |
| [CorsConfig.java](src/main/java/com/yupi/yuaiagent/config/CorsConfig.java) | 跨域配置 | 允许前端跨域访问 |

### 前端（按学习顺序）

| 文件 | 核心内容 | 学习重点 |
|------|----------|----------|
| [router/index.js](yu-ai-agent-frontend/src/router/index.js) | 路由 | SPA 三个页面、导航守卫 |
| [api/index.js](yu-ai-agent-frontend/src/api/index.js) | API 层 | SSE EventSource 封装 |
| [ChatRoom.vue](yu-ai-agent-frontend/src/components/ChatRoom.vue) | 聊天组件 | props/emit 模式、自动滚动 |
| [LoveMaster.vue](yu-ai-agent-frontend/src/views/LoveMaster.vue) | 恋爱大师 | 消息逐步拼接 |
| [SuperAgent.vue](yu-ai-agent-frontend/src/views/SuperAgent.vue) | 超级智能体 | 气泡分割、延迟显示 |
| [Home.vue](yu-ai-agent-frontend/src/views/Home.vue) | 首页 | CSS 动画、赛博朋克风格 |
| [App.vue](yu-ai-agent-frontend/src/App.vue) | 根组件 | 全局样式 + `<router-view>` |
| [main.js](yu-ai-agent-frontend/src/main.js) | 入口 | Vue 应用挂载 |

---

## 学习进度自检

每学完一个模块，试着回答以下问题：

- [ ] 用户点击"发送"后，数据经过了哪些文件，最终到达 AI？
- [ ] SSE 和普通 HTTP 的区别是什么？什么时候用哪个？
- [ ] ChatRoom 为什么能同时被恋爱大师和超级智能体复用？
- [ ] BaseAgent、ReActAgent、ToolCallAgent、YuManus 各自做了什么？
- [ ] think() 和 act() 的区别是什么？为什么要把它们分开？
- [ ] 如果没有 TerminateTool，Agent 会发生什么？
- [ ] RAG 的三个步骤是什么？Query 重写为什么有必要？
- [ ] Spring AI 帮我们省掉了哪些工作？如果不用它，要自己写什么？

全部能回答出来，这个项目你就吃透了。
