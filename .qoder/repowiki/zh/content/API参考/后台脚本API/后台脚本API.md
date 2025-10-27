# 后台脚本API

<cite>
**本文档中引用的文件**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts)
- [context.ts](file://chrome-extension/src/background/browser/context.ts)
- [manager.ts](file://chrome-extension/src/background/agent/event/manager.ts)
- [types.ts](file://chrome-extension/src/background/agent/types.ts)
- [page.ts](file://chrome-extension/src/background/browser/page.ts)
- [views.ts](file://chrome-extension/src/background/browser/views.ts)
- [history.ts](file://chrome-extension/src/background/agent/history.ts)
- [views.ts](file://chrome-extension/src/background/browser/dom/views.ts)
- [service.ts](file://chrome-extension/src/background/agent/messages/service.ts)
- [types.ts](file://chrome-extension/src/background/agent/event/types.ts)
- [errors.ts](file://chrome-extension/src/background/agent/agents/errors.ts)
- [index.ts](file://chrome-extension/src/background/index.ts)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概览](#架构概览)
5. [详细组件分析](#详细组件分析)
6. [依赖关系分析](#依赖关系分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)

## 简介

NanoBrowser是一个基于Chrome扩展的智能网页自动化系统，提供了强大的后台脚本API。该系统通过Executor类协调多个智能代理（Navigator和Planner）完成复杂的网页交互任务，同时提供了完善的浏览器上下文管理和事件驱动架构。

本文档详细介绍了系统的核心API，包括Executor类的构造函数、执行方法、状态管理，BrowserContext类的页面操作和导航控制，以及各个核心类之间的协作机制。

## 项目结构

系统采用模块化架构，主要分为以下几个核心模块：

```mermaid
graph TB
subgraph "后台脚本核心"
Executor[Executor 执行器]
BrowserContext[BrowserContext 浏览器上下文]
EventManager[EventManager 事件管理器]
MessageManager[MessageManager 消息管理器]
end
subgraph "代理系统"
NavigatorAgent[NavigatorAgent 导航代理]
PlannerAgent[PlannerAgent 规划代理]
end
subgraph "浏览器操作"
Page[Page 页面操作]
DOMService[DOMService DOM服务]
Views[Views 视图层]
end
subgraph "支持模块"
Types[Types 类型定义]
Errors[Errors 错误处理]
History[History 历史记录]
end
Executor --> BrowserContext
Executor --> EventManager
Executor --> MessageManager
BrowserContext --> Page
BrowserContext --> DOMService
NavigatorAgent --> Page
PlannerAgent --> MessageManager
EventManager --> Types
MessageManager --> Types
```

**图表来源**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L1-L50)
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L1-L30)
- [manager.ts](file://chrome-extension/src/background/agent/event/manager.ts#L1-L20)

**章节来源**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L1-L435)
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L1-L361)

## 核心组件

### Executor类 - 主执行器

Executor类是整个系统的核心控制器，负责协调所有代理的执行流程。

#### 构造函数

```typescript
constructor(
  task: string,
  taskId: string,
  browserContext: BrowserContext,
  navigatorLLM: BaseChatModel,
  extraArgs?: Partial<ExecutorExtraArgs>,
)
```

**参数说明：**
- `task`: 要执行的任务描述
- `taskId`: 唯一的任务标识符
- `browserContext`: 浏览器上下文实例
- `navigatorLLM`: 导航代理使用的语言模型
- `extraArgs`: 可选的额外参数配置

**配置选项：**
- `plannerLLM`: 规划代理使用的语言模型（可选，默认与导航模型相同）
- `extractorLLM`: 提取器使用的语言模型（可选，默认与导航模型相同）
- `agentOptions`: 代理配置选项
- `generalSettings`: 通用设置配置

#### 核心方法

##### execute()

异步执行任务的主要方法，包含完整的执行生命周期：

```mermaid
flowchart TD
Start([开始执行]) --> InitTask[初始化任务]
InitTask --> ResetCounter[重置步骤计数器]
ResetCounter --> EmitStart[发出TASK_START事件]
EmitStart --> LoopStart{是否达到最大步骤?}
LoopStart --> |否| CheckPause{检查暂停状态}
CheckPause --> |已暂停| WaitResume[等待恢复]
WaitResume --> CheckPause
CheckPause --> |未暂停| CheckStop{检查停止状态}
CheckStop --> |已停止| StopExecution[停止执行]
CheckStop --> |未停止| RunPlanner{需要运行规划器?}
RunPlanner --> |是| ExecutePlanner[执行规划器]
ExecutePlanner --> CheckCompletion[检查任务完成]
CheckCompletion --> |完成| EmitSuccess[发出TASK_OK事件]
CheckCompletion --> |未完成| ExecuteNavigator[执行导航器]
ExecuteNavigator --> CheckNavigatorDone{导航器完成?}
CheckNavigatorDone --> |是| LoopStart
CheckNavigatorDone --> |否| LoopStart
LoopStart --> |是| EmitFailure[发出TASK_FAIL事件]
EmitSuccess --> Cleanup[清理资源]
EmitFailure --> Cleanup
StopExecution --> Cleanup
Cleanup --> StoreHistory[存储历史记录]
StoreHistory --> End([结束])
```

**图表来源**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L120-L200)

##### cancel()

取消当前正在执行的任务：

```typescript
async cancel(): Promise<void>
```

##### pause()

暂停任务执行：

```typescript
async pause(): Promise<void>
```

##### resume()

恢复暂停的任务：

```typescript
async resume(): Promise<void>
```

##### cleanup()

清理浏览器上下文资源：

```typescript
async cleanup(): Promise<void>
```

##### replayHistory()

重放历史操作记录：

```typescript
async replayHistory(
  sessionId: string,
  maxRetries?: number,
  skipFailures?: boolean,
  delayBetweenActions?: number
): Promise<ActionResult[]>
```

**参数说明：**
- `sessionId`: 历史记录会话ID
- `maxRetries`: 最大重试次数（默认3次）
- `skipFailures`: 是否跳过失败的操作
- `delayBetweenActions`: 操作间延迟（秒）

**章节来源**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L40-L435)

### BrowserContext类 - 浏览器上下文管理

BrowserContext类负责管理浏览器窗口、标签页和页面状态。

#### 核心方法

##### getCurrentPage()

获取当前活动页面：

```typescript
async getCurrentPage(): Promise<Page>
```

##### switchTab()

切换到指定标签页：

```typescript
async switchTab(tabId: number): Promise<Page>
```

##### openTab()

打开新标签页：

```typescript
async openTab(url: string): Promise<Page>
```

##### closeTab()

关闭指定标签页：

```typescript
async closeTab(tabId: number): Promise<void>
```

##### navigateTo()

导航到指定URL：

```typescript
async navigateTo(url: string): Promise<void>
```

**章节来源**
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L80-L361)

### AgentContext类 - 代理上下文

AgentContext类为代理提供共享的状态和资源访问。

#### 核心属性

- `controller`: 中止控制器
- `taskId`: 任务唯一标识符
- `browserContext`: 浏览器上下文引用
- `messageManager`: 消息管理器
- `eventManager`: 事件管理器
- `options`: 代理配置选项
- `paused`: 暂停状态标志
- `stopped`: 停止状态标志
- `consecutiveFailures`: 连续失败计数
- `nSteps`: 已执行步骤数
- `actionResults`: 操作结果列表

**章节来源**
- [types.ts](file://chrome-extension/src/background/agent/types.ts#L30-L120)

## 架构概览

系统采用事件驱动的架构模式，通过多个核心类的协作实现复杂的网页自动化功能：

```mermaid
sequenceDiagram
participant User as 用户
participant Executor as Executor
participant BrowserContext as BrowserContext
participant Page as Page
participant Navigator as NavigatorAgent
participant Planner as PlannerAgent
participant EventManager as EventManager
User->>Executor : execute()
Executor->>EventManager : 发出TASK_START事件
Executor->>BrowserContext : 获取当前页面
BrowserContext->>Page : 创建/获取页面
Page-->>BrowserContext : 返回页面实例
loop 执行循环
Executor->>Planner : 执行规划器
Planner-->>Executor : 返回规划结果
Executor->>Navigator : 执行导航器
Navigator->>Page : 执行页面操作
Page-->>Navigator : 返回操作结果
Navigator-->>Executor : 返回导航结果
Executor->>EventManager : 发出STEP_*事件
end
Executor->>EventManager : 发出TASK_OK/TASK_FAIL事件
Executor->>BrowserContext : cleanup()
BrowserContext->>Page : detachPuppeteer()
```

**图表来源**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L120-L200)
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L80-L150)
- [manager.ts](file://chrome-extension/src/background/agent/event/manager.ts#L1-L53)

## 详细组件分析

### Executor类详细分析

#### 执行生命周期管理

Executor类实现了完整的任务执行生命周期，包括初始化、执行、监控和清理阶段：

```mermaid
stateDiagram-v2
[*] --> 初始化
初始化 --> 准备就绪 : 构造函数完成
准备就绪 --> 执行中 : execute()
执行中 --> 暂停 : pause()
执行中 --> 完成 : 任务成功
执行中 --> 失败 : 任务失败
执行中 --> 停止 : cancel()
暂停 --> 执行中 : resume()
完成 --> 清理 : cleanup()
失败 --> 清理 : cleanup()
停止 --> 清理 : cleanup()
清理 --> [*]
```

**图表来源**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L120-L250)

#### 错误处理机制

系统实现了分层的错误处理机制：

```mermaid
flowchart TD
Error[捕获错误] --> CheckType{错误类型判断}
CheckType --> |认证错误| AuthError[ChatModelAuthError]
CheckType --> |权限错误| ForbiddenError[ChatModelForbiddenError]
CheckType --> |请求错误| BadRequestError[ChatModelBadRequestError]
CheckType --> |扩展冲突| ConflictError[ExtensionConflictError]
CheckType --> |其他错误| GenericError[通用错误处理]
AuthError --> LogError[记录错误日志]
ForbiddenError --> LogError
BadRequestError --> LogError
ConflictError --> LogError
GenericError --> LogError
LogError --> CheckRetry{检查重试条件}
CheckRetry --> |可重试| Retry[执行重试]
CheckRetry --> |不可重试| EmitFail[发出失败事件]
Retry --> Success{重试成功?}
Success --> |是| Continue[继续执行]
Success --> |否| MaxFailures{达到最大失败次数?}
MaxFailures --> |是| EmitFail
MaxFailures --> |否| Continue
Continue --> CheckStop{检查停止状态}
CheckStop --> |已停止| EmitCancel[发出取消事件]
CheckStop --> |未停止| NextStep[下一步骤]
```

**图表来源**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L200-L250)
- [errors.ts](file://chrome-extension/src/background/agent/agents/errors.ts#L1-L315)

**章节来源**
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L200-L300)
- [errors.ts](file://chrome-extension/src/background/agent/agents/errors.ts#L1-L100)

### BrowserContext类详细分析

#### 页面管理机制

BrowserContext类实现了智能的页面管理策略：

```mermaid
classDiagram
class BrowserContext {
-_config : BrowserContextConfig
-_currentTabId : number
-_attachedPages : Map~number, Page~
+getConfig() : BrowserContextConfig
+updateConfig(config) : void
+getCurrentPage() : Promise~Page~
+switchTab(tabId) : Promise~Page~
+openTab(url) : Promise~Page~
+closeTab(tabId) : Promise~void~
+navigateTo(url) : Promise~void~
+getState(useVision) : Promise~BrowserState~
+getCachedState() : Promise~BrowserState~
}
class Page {
-_tabId : number
-_browser : Browser
-_puppeteerPage : PuppeteerPage
-_state : PageState
+attachPuppeteer() : Promise~boolean~
+detachPuppeteer() : Promise~void~
+navigateTo(url) : Promise~void~
+getState(useVision) : Promise~PageState~
+takeScreenshot() : Promise~string~
}
class BrowserState {
+elementTree : DOMElementNode
+selectorMap : Map
+tabId : number
+url : string
+title : string
+screenshot : string
+tabs : TabInfo[]
}
BrowserContext --> Page : manages
BrowserContext --> BrowserState : creates
Page --> BrowserState : generates
```

**图表来源**
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L15-L100)
- [page.ts](file://chrome-extension/src/background/browser/page.ts#L50-L150)
- [views.ts](file://chrome-extension/src/background/browser/views.ts#L80-L120)

#### 标签页操作流程

```mermaid
flowchart TD
SwitchTab[switchTab] --> UpdateTab[更新标签页激活状态]
UpdateTab --> WaitEvents[等待标签页事件]
WaitEvents --> GetTab[获取标签页信息]
GetTab --> GetOrCreatePage[获取或创建页面]
GetOrCreatePage --> AttachPage[附加页面]
AttachPage --> UpdateCurrent[更新当前标签页ID]
UpdateCurrent --> ReturnPage[返回页面实例]
OpenTab[openTab] --> CreateTab[创建新标签页]
CreateTab --> WaitTabEvents[等待标签页事件]
WaitTabEvents --> GetUpdatedTab[获取更新后的标签页]
GetUpdatedTab --> CreatePage[创建页面实例]
CreatePage --> AttachNewPage[附加新页面]
AttachNewPage --> UpdateCurrentID[更新当前标签页ID]
UpdateCurrentID --> ReturnNewPage[返回新页面]
```

**图表来源**
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L200-L280)

**章节来源**
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L200-L361)

### EventManager类详细分析

EventManager类实现了事件驱动的通信机制：

#### 事件订阅和发布

```mermaid
classDiagram
class EventManager {
-_subscribers : Map~EventType, EventCallback[]~
+subscribe(eventType, callback) : void
+unsubscribe(eventType, callback) : void
+clearSubscribers(eventType) : void
+emit(event) : Promise~void~
}
class AgentEvent {
+actor : Actors
+state : ExecutionState
+data : EventData
+timestamp : number
+type : EventType
}
class EventCallback {
<<interface>>
+callback(event) : Promise~void~
}
EventManager --> AgentEvent : emits
EventManager --> EventCallback : invokes
```

**图表来源**
- [manager.ts](file://chrome-extension/src/background/agent/event/manager.ts#L1-L53)
- [types.ts](file://chrome-extension/src/background/agent/event/types.ts#L60-L78)

**章节来源**
- [manager.ts](file://chrome-extension/src/background/agent/event/manager.ts#L1-L53)

### MessageManager类详细分析

MessageManager类负责管理与语言模型的对话历史：

#### 消息管理流程

```mermaid
flowchart TD
InitTask[初始化任务消息] --> AddSystem[添加系统消息]
AddSystem --> AddContext[添加上下文消息]
AddContext --> AddTask[添加任务指令]
AddTask --> AddSensitive[添加敏感数据信息]
AddSensitive --> AddExample[添加示例输出]
AddExample --> AddHistoryMarker[添加历史标记]
AddNewTask[添加新任务] --> FilterTask[过滤任务内容]
FilterTask --> WrapTask[包装任务请求]
WrapTask --> AddTaskMessage[添加任务消息]
AddPlan[添加规划消息] --> FilterPlan[过滤规划内容]
FilterPlan --> WrapPlan[包装规划内容]
WrapPlan --> AddPlanMessage[添加规划消息]
AddModelOutput[添加模型输出] --> CreateToolCall[创建工具调用]
CreateToolCall --> AddAIMessage[添加AI消息]
AddAIMessage --> AddToolMessage[添加工具响应]
```

**图表来源**
- [service.ts](file://chrome-extension/src/background/agent/messages/service.ts#L50-L150)

**章节来源**
- [service.ts](file://chrome-extension/src/background/agent/messages/service.ts#L1-L441)

## 依赖关系分析

系统的依赖关系体现了清晰的分层架构：

```mermaid
graph TD
subgraph "应用层"
BackgroundScript[background.ts]
end
subgraph "业务逻辑层"
Executor[Executor]
BrowserContext[BrowserContext]
EventManager[EventManager]
MessageManager[MessageManager]
end
subgraph "代理层"
NavigatorAgent[NavigatorAgent]
PlannerAgent[PlannerAgent]
end
subgraph "浏览器操作层"
Page[Page]
DOMService[DOMService]
Puppeteer[Puppeteer Core]
end
subgraph "基础设施层"
Types[Types]
Errors[Errors]
History[History]
Views[Views]
end
BackgroundScript --> Executor
Executor --> BrowserContext
Executor --> EventManager
Executor --> MessageManager
BrowserContext --> Page
BrowserContext --> DOMService
Page --> Puppeteer
NavigatorAgent --> Page
PlannerAgent --> MessageManager
EventManager --> Types
MessageManager --> Types
Executor --> Errors
Executor --> History
BrowserContext --> Views
```

**图表来源**
- [index.ts](file://chrome-extension/src/background/index.ts#L1-L50)
- [executor.ts](file://chrome-extension/src/background/agent/executor.ts#L1-L30)
- [context.ts](file://chrome-extension/src/background/browser/context.ts#L1-L20)

**章节来源**
- [index.ts](file://chrome-extension/src/background/index.ts#L1-L100)

## 性能考虑

### 状态缓存机制

系统实现了多层缓存机制以提高性能：

1. **页面状态缓存**: Page类维护了页面状态的缓存，避免重复计算
2. **DOM元素哈希缓存**: 使用哈希值快速比较DOM元素变化
3. **历史记录缓存**: 支持历史任务的重放和缓存

### 内存管理

- **自动清理**: 执行完成后自动清理浏览器上下文
- **连接池管理**: Puppeteer连接的智能管理
- **消息历史限制**: 消息历史的令牌限制和自动截断

### 并发控制

- **事件队列**: 事件管理器使用队列确保事件顺序
- **异步操作**: 所有长时间操作都使用异步模式
- **超时控制**: 关键操作设置了合理的超时时间

## 故障排除指南

### 常见错误及解决方案

#### 认证错误处理

当遇到认证相关错误时，系统会：
1. 检查API密钥的有效性
2. 验证权限设置
3. 提供详细的错误信息和解决建议

#### 扩展冲突问题

当检测到扩展冲突时：
1. 显示友好的错误提示
2. 建议在新配置文件中使用
3. 自动清理冲突资源

#### 网页加载超时

对于网页加载超时：
1. 实现智能等待策略
2. 提供网络空闲检测
3. 设置最大等待时间限制

**章节来源**
- [errors.ts](file://chrome-extension/src/background/agent/agents/errors.ts#L100-L200)

### 性能优化建议

1. **合理设置超时时间**: 根据目标网站特性调整等待时间
2. **启用视觉模式**: 对于复杂界面启用视觉识别
3. **优化令牌使用**: 合理配置消息历史长度
4. **使用状态缓存**: 充分利用内置的缓存机制

## 结论

NanoBrowser的后台脚本API提供了一个完整而强大的网页自动化解决方案。通过Executor类的统一调度、BrowserContext类的智能页面管理、EventManger类的事件驱动架构，以及各代理的专业化分工，系统能够高效地完成复杂的网页交互任务。

系统的设计充分考虑了可扩展性、可靠性和易用性，为开发者提供了丰富的API接口和完善的错误处理机制。通过合理的配置和使用，可以满足各种复杂的网页自动化需求。