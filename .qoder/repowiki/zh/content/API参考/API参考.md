# API参考

<cite>
**本文档中引用的文件**  
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts)
- [context.ts](file://chrome-extension/src/background/browser/context.ts)
- [page.ts](file://chrome-extension/src/background/browser/page.ts)
- [service.ts](file://chrome-extension/src/background/agent/messages/service.ts)
- [manager.ts](file://chrome-extension/src/background/agent/event/manager.ts)
- [types.ts](file://chrome-extension/src/background/agent/event/types.ts)
- [base.ts](file://packages/storage/lib/base/base.ts)
- [index.ts](file://packages/storage/lib/index.ts)
- [json_schema.ts](file://packages/schema-utils/lib/json_schema.ts)
</cite>

## 目录
1. [核心类API](#核心类api)
2. [消息系统](#消息系统)
3. [共享包接口](#共享包接口)
4. [事件类型与状态](#事件类型与状态)

## 核心类API

### Executor类
Executor类是任务执行的核心控制器，负责协调规划器和导航器代理来完成用户任务。

**Section sources**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L1-L435)

#### 构造函数
```typescript
constructor(
  task: string,
  taskId: string,
  browserContext: BrowserContext,
  navigatorLLM: BaseChatModel,
  extraArgs?: Partial<ExecutorExtraArgs>
)
```
初始化Executor实例，设置任务描述、浏览器上下文和语言模型。

#### 公共方法
- `execute()`: 执行当前任务，包含规划和导航循环
- `cancel()`: 取消当前任务执行
- `pause()`: 暂停任务执行
- `resume()`: 恢复暂停的任务
- `addFollowUpTask(task: string)`: 添加后续任务
- `replayHistory(sessionId: string, maxRetries?: number, skipFailures?: boolean, delayBetweenActions?: number)`: 重放指定会话的历史任务
- `subscribeExecutionEvents(callback: EventCallback)`: 订阅执行事件
- `clearExecutionEvents()`: 清除所有执行事件订阅
- `cleanup()`: 清理执行器资源

### BrowserContext类
BrowserContext类管理浏览器标签页和页面状态，提供标签页切换、导航和状态获取功能。

**Section sources**
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L1-L361)

#### 构造函数
```typescript
constructor(config: Partial<BrowserContextConfig>)
```
使用指定配置初始化浏览器上下文。

#### 公共方法
- `switchTab(tabId: number)`: 切换到指定标签页
- `openTab(url: string)`: 打开新标签页
- `closeTab(tabId: number)`: 关闭指定标签页
- `navigateTo(url: string)`: 导航到指定URL
- `getCurrentPage()`: 获取当前页面实例
- `getState(useVision?: boolean, cacheClickableElementsHashes?: boolean)`: 获取当前浏览器状态
- `getCachedState(useVision?: boolean, cacheClickableElementsHashes?: boolean)`: 获取缓存的浏览器状态
- `cleanup()`: 清理浏览器上下文资源
- `removeHighlight()`: 移除页面高亮元素

### Page类
Page类封装了单个标签页的操作，包括DOM操作、截图和滚动控制。

**Section sources**
- [page.ts](file://chrome-extension/src/background/browser/page.ts#L1-L799)

#### 构造函数
```typescript
constructor(tabId: number, url: string, title: string, config: Partial<BrowserContextConfig> = {})
```
初始化页面实例，关联到指定标签页。

#### 公共方法
- `attachPuppeteer()`: 连接Puppeteer调试协议
- `detachPuppeteer()`: 断开Puppeteer连接
- `getState(useVision?: boolean, cacheClickableElementsHashes?: boolean)`: 获取页面状态
- `takeScreenshot(fullPage?: boolean)`: 截取页面截图
- `navigateTo(url: string)`: 导航到指定URL
- `scrollToPercent(yPercent: number, elementNode?: DOMElementNode)`: 滚动到页面指定百分比位置
- `scrollBy(y: number, elementNode?: DOMElementNode)`: 按指定像素值滚动
- `sendKeys(keys: string)`: 向页面发送键盘输入
- `getClickableElements(showHighlightElements: boolean, focusElement: number)`: 获取可点击元素
- `removeHighlight()`: 移除页面高亮

## 消息系统

### MessageManager类
MessageManager类负责管理对话消息历史，包括消息的添加、过滤和令牌计数。

**Section sources**
- [service.ts](file://chrome-extension/src/background/agent/messages/service.ts#L1-L441)

#### 构造函数
```typescript
constructor(settings: MessageManagerSettings = new MessageManagerSettings())
```
使用指定设置初始化消息管理器。

#### 公共方法
- `initTaskMessages(systemMessage: SystemMessage, task: string, messageContext?: string)`: 初始化任务消息历史
- `addNewTask(newTask: string)`: 添加新任务到消息历史
- `addPlan(plan?: string, position?: number)`: 添加规划消息
- `addStateMessage(stateMessage: HumanMessage)`: 添加状态消息
- `addModelOutput(modelOutput: Record<string, unknown>)`: 添加模型输出消息
- `addToolMessage(content: string, toolCallId?: number, messageType?: string | null)`: 添加工具调用消息
- `getMessages()`: 获取所有消息
- `cutMessages()`: 根据令牌限制裁剪消息
- `length()`: 获取消息数量

#### MessageManagerSettings类
配置消息管理器的行为参数。

##### 属性
- `maxInputTokens`: 最大输入令牌数
- `estimatedCharactersPerToken`: 每个令牌的估计字符数
- `imageTokens`: 图像令牌数
- `includeAttributes`: 包含的属性列表
- `messageContext`: 消息上下文
- `sensitiveData`: 敏感数据映射
- `availableFilePaths`: 可用文件路径列表

## 共享包接口

### @extension/storage包
提供浏览器存储功能的封装，支持本地存储和会话存储。

**Section sources**
- [base.ts](file://packages/storage/lib/base/base.ts#L1-L158)
- [index.ts](file://packages/storage/lib/index.ts#L1-L9)

#### createStorage函数
```typescript
function createStorage<D = string>(key: string, fallback: D, config?: StorageConfig<D>): BaseStorage<D>
```
创建存储实例。

##### 参数
- `key`: 存储键名
- `fallback`: 默认值
- `config`: 存储配置

##### 返回值
`BaseStorage<D>`: 存储实例，提供get、set、subscribe等方法

#### BaseStorage接口
```typescript
interface BaseStorage<D> {
  get(): Promise<D>;
  set(valueOrUpdate: ValueOrUpdate<D>): Promise<void>;
  getSnapshot(): D | null;
  subscribe(listener: () => void): () => void;
}
```

### @extension/schema-utils包
提供JSON模式操作工具。

**Section sources**
- [json_schema.ts](file://packages/schema-utils/lib/json_schema.ts#L1-L5)

#### 主要导出
- `json_schema`: JSON模式操作工具
- `json_gemini`: Gemini格式JSON工具
- `helpers`: 辅助函数
- `helper`: 单个辅助工具

## 事件类型与状态

### 事件系统
事件系统用于在组件间传递状态变化信息。

**Section sources**
- [manager.ts](file://chrome-extension/src/background/agent/event/manager.ts#L1-L53)
- [types.ts](file://chrome-extension/src/background/agent/event/types.ts#L1-L78)

### EventManager类
管理事件订阅和发布。

#### 公共方法
- `subscribe(eventType: EventType, callback: EventCallback)`: 订阅事件
- `unsubscribe(eventType: EventType, callback: EventCallback)`: 取消订阅
- `clearSubscribers(eventType: EventType)`: 清除指定类型的订阅
- `emit(event: AgentEvent)`: 发布事件

### 事件类型枚举

#### Actors
表示事件的发起者：
- `SYSTEM`: 系统
- `USER`: 用户
- `PLANNER`: 规划器
- `NAVIGATOR`: 导航器

#### EventType
事件类型：
- `EXECUTION`: 执行事件

#### ExecutionState
执行状态，格式为`<作用域>.<状态>`：
- **任务级别状态**:
  - `TASK_START`: 任务开始
  - `TASK_OK`: 任务成功
  - `TASK_FAIL`: 任务失败
  - `TASK_PAUSE`: 任务暂停
  - `TASK_RESUME`: 任务恢复
  - `TASK_CANCEL`: 任务取消
- **步骤级别状态**:
  - `STEP_START`: 步骤开始
  - `STEP_OK`: 步骤成功
  - `STEP_FAIL`: 步骤失败
  - `STEP_CANCEL`: 步骤取消
- **动作/工具级别状态**:
  - `ACT_START`: 动作开始
  - `ACT_OK`: 动作成功
  - `ACT_FAIL`: 动作失败

### AgentEvent类
表示执行系统中的状态变化事件。

#### 构造函数
```typescript
constructor(
  actor: Actors,
  state: ExecutionState,
  data: EventData,
  timestamp?: number,
  type?: EventType
)
```

#### 数据结构
```typescript
interface EventData {
  taskId: string;           // 任务ID
  step: number;             // 事件发生的步骤号
  maxSteps: number;         // 任务最大步骤数
  details: string;          // 事件内容
}
```