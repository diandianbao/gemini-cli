# Gemini CLI 详细序列图（包含文件路径和行号）

## 主启动序列

### 序列图 1: CLI 启动流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant Index as 入口文件(index.ts)
    participant Gemini as 主模块(gemini.tsx:294)
    participant Settings as 设置管理(settings.ts)
    participant ConfigParse as 参数解析(config.ts:94)
    participant ConfigLoad1 as 配置加载1(config.ts:416)
    participant Auth as 认证处理(auth.ts)
    participant Sandbox as 沙箱处理(sandbox.ts)
    participant ConfigLoad2 as 配置加载2(config.ts:416)
    participant Initializer as 应用初始化(initializer.ts:35)
    participant UI as UI启动(gemini.tsx:179)
    participant NonInteractive as 非交互模式(nonInteractiveCli.ts:58)
    participant Cleanup as 清理处理(cleanup.ts)

    User->>Index: 执行 gemini 命令
    Index->>Gemini: 调用 main() 函数 (gemini.tsx:294)

    Note over Gemini: 设置启动清理和监听器
    Gemini->>Gemini: patchStdio() / setupUnhandledRejectionHandler() (gemini.tsx:304-311)

    Note over Gemini,Settings: 第一阶段：加载基础设置和参数
    Gemini->>Settings: 加载设置 loadSettings() (gemini.tsx:313)
    Settings-->>Gemini: 返回 LoadedSettings

    Gemini->>Settings: 加载信任文件夹 loadTrustedFolders() (gemini.tsx:321)
    Gemini->>Gemini: 清理过期资源 (gemini.tsx:329-332)

    Gemini->>ConfigParse: 解析参数 parseArguments() (gemini.tsx:335)
    ConfigParse-->>Gemini: 返回 CliArgs

    Note over Gemini,Auth: 第二阶段：初步配置和认证
    Gemini->>ConfigLoad1: 初次加载配置 loadCliConfig() (gemini.tsx:385)
    ConfigLoad1-->>Gemini: 返回 partialConfig

    alt 交互模式且未使用外部认证
        Gemini->>Auth: validateAuthMethod() (gemini.tsx:400)
        Gemini->>Auth: refreshAuth() (gemini.tsx:407)
        Auth-->>Gemini: 认证结果
    else 非交互模式
        Gemini->>Auth: validateNonInteractiveAuth() (gemini.tsx:411)
        Gemini->>Auth: refreshAuth() (gemini.tsx:417)
        Auth-->>Gemini: 认证结果
    end

    Gemini->>Gemini: 加载远程管理设置 (gemini.tsx:436-440)
    Gemini->>Gemini: 执行延迟命令 runDeferredCommand() (gemini.tsx:443)

    Note over Gemini,Sandbox: 第三阶段：沙箱检查
    alt 不在沙箱中 && 启用了沙箱配置
        Gemini->>Sandbox: loadSandboxConfig() (gemini.tsx:450)
        Sandbox-->>Gemini: 返回 sandboxConfig
        Gemini->>Gemini: start_sandbox() (gemini.tsx:493)
        Note over Gemini: 沙箱启动后会重新运行 main()
        Gemini->>Cleanup: runExitCleanup() (gemini.tsx:495)
        Gemini->>User: 退出并重启到沙箱
    else 不在沙箱中 && 未启用沙箱
        Gemini->>Gemini: relaunchAppInChildProcess() (gemini.tsx:500)
        Note over Gemini: 在子进程中重新运行
    else 在沙箱中（直接运行）
        Note over Gemini,ConfigLoad2: 第四阶段：最终配置加载

        Gemini->>ConfigLoad2: 完整加载配置 loadCliConfig() (gemini.tsx:509)
        ConfigLoad2-->>Gemini: 返回 Config 对象

        Note over Gemini: 设置遥测、策略、会话清理
        Gemini->>Gemini: 注册遥历和策略 (gemini.tsx:526-541)

        alt --list-extensions
            Gemini->>Gemini: 列出扩展并退出 (gemini.tsx:543-549)
        else --list-sessions
            Gemini->>Gemini: 列出会话并退出 (gemini.tsx:553-571)
        else --delete-session
            Gemini->>Gemini: 删除会话并退出 (gemini.tsx:574-579)
        end

        Note over Gemini: 设置终端模式和主题
        Gemini->>Gemini: 设置 stdin Raw Mode (gemini.tsx:582-606)
        Gemini->>Gemini: setupTerminalAndTheme() (gemini.tsx:608)

        Note over Gemini,Initializer: 第五阶段：应用初始化
        Gemini->>Initializer: initializeApp() (gemini.tsx:611)
        Initializer->>Auth: performInitialAuth() (initializer.ts:40)
        Initializer->>Auth: validateTheme() (initializer.ts:45)
        Initializer-->>Gemini: 返回 InitializationResult

        alt Zed 集成模式
            Gemini->>Gemini: runZedIntegration() (gemini.tsx:624)
        else 交互模式 (config.isInteractive())
            Note over Gemini: 处理 --resume 参数
            alt --resume
                Gemini->>Gemini: SessionSelector.resolveSession() (gemini.tsx:636-653)
            end

            Gemini->>UI: startInteractiveUI() (gemini.tsx:658)
            UI->>Gemini: 渲染 React 应用 (ink render)
            UI->>UI: checkForUpdates() (gemini.tsx:280)
        else 非交互模式
            Gemini->>Gemini: config.initialize() (gemini.tsx:669)
            Gemini->>Gemini: 读取 stdin 数据 (gemini.tsx:674-680)

            Note over Gemini: 触发 SessionStart 钩子
            Gemini->>Gemini: hookSystem.fireSessionStartEvent() (gemini.tsx:690)

            Gemini->>NonInteractive: runNonInteractive() (gemini.tsx:743)
            NonInteractive->>NonInteractive: 创建 Scheduler (nonInteractiveCli.ts:213)
            NonInteractive->>NonInteractive: 处理用户输入循环 (nonInteractiveCli.ts:290-517)
            NonInteractive-->>Gemini: 执行完成
        end
    end

    Gemini->>Cleanup: runExitCleanup() (gemini.tsx:751)
    Cleanup->>User: 程序退出
```

## 配置加载详细序列

### 序列图 2: loadCliConfig() 详细流程

```mermaid
sequenceDiagram
    participant Caller as 调用者
    participant LoadConfig as loadCliConfig(config.ts:416)
    participant Settings as 设置(loadSettings)
    participant Env as 环境设置
    participant FileService as 文件服务(FileDiscoveryService)
    participant ExtMgr as 扩展管理器(ExtensionManager)
    participant MemTool as 内存加载(loadServerHierarchicalMemory)
    participant Policy as 策略(createPolicyEngineConfig)
    participant SandboxConfig as 沙箱配置
    participant ConfigCtor as Config构造函数

    Caller->>LoadConfig: loadCliConfig(settings, sessionId, argv, options)

    Note over LoadConfig,Settings: 第1步：加载环境和工作目录设置
    LoadConfig->>Settings: loadSettings(cwd) (config.ts:425)
    Settings-->>LoadConfig: 返回 loadedSettings

    alt argv.sandbox
        LoadConfig->>Env: 设置 GEMINI_SANDBOX=true (config.ts:428)
    end

    Note over LoadConfig,FileService: 第2步：准备文件服务和配置
    LoadConfig->>FileService: new FileDiscoveryService(cwd) (config.ts:449)

    Note over LoadConfig,ExtMgr: 第3步：创建和加载扩展
    LoadConfig->>ExtMgr: new ExtensionManager({...}) (config.ts:465-473)
    ExtMgr-->>LoadConfig: 返回 extensionManager
    LoadConfig->>ExtMgr: extensionManager.loadExtensions() (config.ts:474)
    ExtMgr-->>LoadConfig: 扩展加载完成

    Note over LoadConfig,MemTool: 第4步：加载内存/上下文
    alt 非实验性 JIT 上下文
        LoadConfig->>MemTool: loadServerHierarchicalMemory() (config.ts:484-499)
        LoadConfig->>FileService: 文件发现服务
        Mem-->>LoadConfig: memoryContent, fileCount, filePaths
    else 实验性 JIT 上下文
        Note over LoadConfig: 跳过内存加载 (experimentalJitContext=true)
    end

    Note over LoadConfig,Policy: 第5步：确定批准模式
    LoadConfig->>LoadConfig: 确定 approvalMode (config.ts:505-569)
    alt 需要 YOLO 模式但被禁用
        LoadConfig->>LoadConfig: 抛出 FatalConfigError
    end

    Note over LoadConfig: 第6步：遥测设置
    LoadConfig->>LoadConfig: resolveTelemetrySettings() (config.ts:573-584)

    Note over LoadConfig: 第7步：确定交互模式
    LoadConfig->>LoadConfig: 确定 interactive 标志 (config.ts:588-591)

    Note over LoadConfig: 第8步：构建工具排除列表
    LoadConfig->>LoadConfig: createToolExclusionFilter() (config.ts:597-633)
    LoadConfig->>LoadConfig: mergeExcludeTools() (config.ts:635)

    Note over LoadConfig,Policy: 第9步：创建策略引擎
    LoadConfig->>Policy: createPolicyEngineConfig() (config.ts:651-654)
    Policy-->>LoadConfig: policyEngineConfig

    Note over LoadConfig,SandboxConfig: 第10步：获取其他配置
    LoadConfig->>SandboxConfig: loadSandboxConfig() (config.ts:667)
    LoadConfig->>LoadConfig: getPty() (config.ts:673)
    LoadConfig->>LoadConfig: McpServerEnablementManager (config.ts:680-683)

    Note over LoadConfig,ConfigCtor: 第11步：创建 Config 对象
    LoadConfig->>ConfigCtor: new Config({...}) (config.ts:685-808)
    ConfigCtor-->>LoadConfig: 返回 Config 实例

    LoadConfig-->>Caller: 返回 Config 对象
```

## 非交互模式执行详细序列

### 序列图 3: runNonInteractive() 详细流程

```mermaid
sequenceDiagram
    participant Caller as 调用者
    participant RunNI as runNonInteractive(nonInteractiveCli.ts:58)
    participant ConsolePatcher as 控制台补丁
    participant OutputFormat as 输出格式器
    participant Scheduler as 调度器(Scheduler)
    participant GeminiClient as Gemini客户端
    participant SlashCmd as 斜杠命令处理器
    participant AtCmd as @命令处理器
    participant AbortCtrl as AbortController

    Caller->>RunNI: runNonInteractive({config, settings, input, prompt_id})

    Note over RunNI,ConsolePatcher: 第1步：初始化
    RunNI->>ConsolePatcher: new ConsolePatcher() (nonInteractiveCli.ts:66-72)
    RunNI->>RunNI: registerActivityLogger() 如果启用 (nonInteractiveCli.ts:74-79)
    RunNI->>OutputFormat: 创建 TextOutput (nonInteractiveCli.ts:81-82)
    RunNI->>AbortCtrl: new AbortController() (nonInteractiveCli.ts:102)

    Note over RunNI: 第2步：设置 stdin 取消监听
    RunNI->>RunNI: setupStdinCancellation() (nonInteractiveCli.ts:112-159)
    Note over RunNI: 设置 Ctrl+C 检测和 AbortController

    Note over RunNI,OutputFormat: 第3步：创建调度器和客户端
    RunNI->>Scheduler: new Scheduler({...}) (nonInteractiveCli.ts:213-218)
    RunNI->>GeminiClient: config.getGeminiClient() (nonInteractiveCli.ts:212)

    alt 恢复会话
        RunNI->>GeminiClient: resumeChat(convertSessionToHistoryFormats()) (nonInteractiveCli.ts:222-228)
    end

    Note over RunNI: 第4步：发送初始化事件(流式JSON)
    alt 使用 STREAM_JSON 输出格式
        RunNI->>OutputFormat: emitEvent(INIT) (nonInteractiveCli.ts:231-238)
    end

    Note over RunNI,SlashCmd: 第5步：处理输入
    alt 输入是斜杠命令
        RunNI->>SlashCmd: handleSlashCommand() (nonInteractiveCli.ts:242-255)
        SlashCmd-->>RunNI: 返回 query(Part[])
    else 输入不是斜杠命令
        RunNI->>AtCmd: handleAtCommand() (nonInteractiveCli.ts:258-275)
        AtCmd-->>RunNI: 返回 processedQuery(Part[])
        alt 处理出错
            RunNI->>RunNI: 抛出 FatalInputError (nonInteractiveCli.ts:270-272)
        end
    end

    RunNI->>RunNI: 发送用户消息事件 (nonInteractiveCli.ts:278-285)

    Note over RunNI,GeminiClient: 第6步：主执行循环
    loop 对话轮次
        RunNI->>RunNI: 检查最大轮次 (nonInteractiveCli.ts:292-297)

        RunNI->>GeminiClient: sendMessageStream(..., abortController.signal) (nonInteractiveCli.ts:300-307)

        Note over RunNI: 第7步：处理流式响应
        loop 流式事件
            GeminiClient-->>RunNI: GeminiEventType
            alt 检查到取消信号
                RunNI->>RunNI: handleCancellationError() (nonInteractiveCli.ts:312)
            end

            alt 事件类型: Content
                RunNI->>RunNI: 输出内容 (nonInteractiveCli.ts:315-333)
            else 事件类型: ToolCallRequest
                RunNI->>RunNI: 收集到 toolCallRequests (nonInteractiveCli.ts:334-344)
                alt 使用 STREAM_JSON
                    RunNI->>OutputFormat: emitEvent(TOOL_USE) (nonInteractiveCli.ts:336-342)
                end
            else 事件类型: LoopDetected
                RunNI->>RunNI: 记录循环检测 (nonInteractiveCli.ts:345-353)
            else 事件类型: MaxSessionTurns
                RunNI->>RunNI: 记录超轮次 (nonInteractiveCli.ts:354-362)
            else 事件类型: Error
                RunNI->>RunNI: 抛出错误 (nonInteractiveCli.ts:364)
            else 事件类型: AgentExecutionStopped
                RunNI->>RunNI: 处理Agent停止 (nonInteractiveCli.ts:365-384)
                RunNI->>RunNI: 跳出主循环
            else 事件类型: AgentExecutionBlocked
                RunNI->>RunNI: 处理Agent阻塞 (nonInteractiveCli.ts:385-390)
            end
        end

        Note over RunNI,Scheduler: 第8步：处理工具调用
        alt 有工具调用请求
            RunNI->>RunNI: textOutput.ensureTrailingNewline() (nonInteractiveCli.ts:394)
            RunNI->>Scheduler: schedule(toolCallRequests, signal) (nonInteractiveCli.ts:395-398)
            Scheduler-->>RunNI: completedToolCalls[]

            Note over RunNI: 处理每个完成的工具调用
            loop 遍历 completedToolCalls
                alt 使用 STREAM_JSON
                    RunNI->>OutputFormat: emitEvent(TOOL_RESULT) (nonInteractiveCli.ts:405-422)
                end

                alt 工具执行出错
                    RunNI->>RunNI: handleToolError() (nonInteractiveCli.ts:426-434)
                end

                alt 工具有响应部分
                    RunNI->>RunNI: 收集 toolResponseParts (nonInteractiveCli.ts:437-440)
                end
            end

            RunNI->>RunNI: recordToolCallInteractions() (nonInteractiveCli.ts:450-455)

            alt 有停止执行工具
                RunNI->>RunNI: 输出停止消息 (nonInteractiveCli.ts:463-467)
                RunNI->>RunNI: 跳出主循环
            end

            RunNI->>GeminiClient: 准准备下一轮消息 (nonInteractiveCli.ts:494)
        else 无工具调用
            RunNI->>RunNI: 输出最终结果 (nonInteractiveCli.ts:496-514)
            RunNI->>RunNI: 跳出主循环
        end
    end

    Note over RunNI: 第9步：清理
    RunNI->>RunNI: cleanupStdinCancellation() (nonInteractiveCli.ts:522)
    RunNI->>ConsolePatcher: cleanup() (nonInteractiveCli.ts:524)

    alt 有错误发生
        RunNI->>RunNI: handleError(error) (nonInteractiveCli.ts:529)
    end

    RunNI-->>Caller: 返回 void
```

## 工具调用详细序列

### 序列图 4: 调度器工具调用流程

```mermaid
sequenceDiagram
  participant Caller as "调用者 (nonInteractiveCli.ts:395)"
  participant Scheduler as "调度器 (Scheduler类)"
  participant StateMgr as "状态管理 (SchedulerStateManager)"
  participant ToolRegistry as "工具注册表 (ToolRegistry)"
  participant Policy as "策略检查 (checkPolicy)"
  participant Confirmation as "确认处理 (resolveConfirmation)"
  participant ToolExecutor as "工具执行器 (ToolExecutor)"

  Caller->>Scheduler: "schedule(toolCallRequests, signal) (scheduler.ts:141)"

  alt 正在处理或已有活动调用
    Scheduler->>Scheduler: "_enqueueRequest() (scheduler.ts:160)"
    Note over Scheduler: 将请求加入队列等待
  else 空闲状态
    Scheduler->>Scheduler: "_startBatch(requests, signal) (scheduler.ts:231)"
  end

  Note over Scheduler,StateMgr: 阶段1：摄入和解析
  Scheduler->>Scheduler: 清理批次状态 (scheduler.ts:237)

  loop 每个工具调用请求
    Scheduler->>ToolRegistry: "getTool(request.name) (scheduler.ts:247)"
    alt 工具未找到
      Scheduler->>Scheduler: "创建错误响应 (scheduler.ts:250-254)"
    else 工具存在
      Scheduler->>Scheduler: "_validateAndCreateToolCall() (scheduler.ts:287)"
      Scheduler->>Scheduler: "tool.build(request.args) (scheduler.ts:299)"
    end
  end

  Scheduler->>StateMgr: "enqueue(newCalls) (scheduler.ts:259)"

  Note over Scheduler,StateMgr: 阶段2：处理循环
  Scheduler->>Scheduler: "_processQueue(signal) (scheduler.ts:328)"

  loop 处理队列中的项目
    Scheduler->>Scheduler: "_processNextItem(signal) (scheduler.ts:339)"

    break 信号已中止或取消
      Scheduler->>StateMgr: "cancelAllQueued() (scheduler.ts:341)"
    end

    alt 无活动调用
      Scheduler->>StateMgr: "dequeue() (scheduler.ts:346)"
      alt 状态为错误
        Scheduler->>StateMgr: "updateStatus(error) (scheduler.ts:350)"
        Scheduler->>StateMgr: "finalizeCall() (scheduler.ts:351)"
      end
    end

    Note over Scheduler,Policy: 阶段3：单个调用编排
    alt 有活动调用且状态为 validating
      Scheduler->>Scheduler: "_processValidatingCall(active, signal) (scheduler.ts:360)"
      Scheduler->>Scheduler: "_processToolCall(active, signal) (scheduler.ts:371)"

      Note over Scheduler,Policy: 步骤1：策略和安全检查
      Scheduler->>Policy: "checkPolicy(toolCall, config) (scheduler.ts:407)"
      Policy-->>Scheduler: 返回 {decision, rule}

      break 决策为 DENY
        Scheduler->>Scheduler: "getPolicyDenialError() (scheduler.ts:410)"
        Scheduler->>StateMgr: "updateStatus(error) (scheduler.ts:415)"
        Scheduler->>StateMgr: "finalizeCall() (scheduler.ts:424)"
      end

      Note over Scheduler,Confirmation: 步骤2：用户确认循环
      alt 决策: ASK_USER
        Scheduler->>Confirmation: "resolveConfirmation() (scheduler.ts:433)"
        Confirmation-->>Scheduler: 返回 {outcome, lastDetails}
      else 决策: APPROVE
        Note over Scheduler: 自动设置 outcome = ProceedOnce
        Scheduler->>StateMgr: "setOutcome(ProceedOnce) (scheduler.ts:444)"
      end

      Note over Scheduler: 步骤3：策略更新
      Scheduler->>Scheduler: "updatePolicy() (scheduler.ts:448)"

      break 用户取消
        Scheduler->>StateMgr: "updateStatus(cancelled) (scheduler.ts:455)"
        Scheduler->>StateMgr: "finalizeCall() (scheduler.ts:456)"
        Scheduler->>StateMgr: "cancelAllQueued() (scheduler.ts:457)"
      end

      Note over Scheduler,ToolExecutor: 步骤4：执行
      Scheduler->>Scheduler: "_execute(callId, signal) (scheduler.ts:462)"
      Scheduler->>StateMgr: "updateStatus(scheduled) (scheduler.ts:471)"
      Scheduler->>StateMgr: "updateStatus(executing) (scheduler.ts:473)"

      Scheduler->>ToolExecutor: "execute({call, signal, outputUpdateHandler}) (scheduler.ts:484)"
      ToolExecutor->>ToolExecutor: 实际执行工具逻辑
      ToolExecutor-->>Scheduler: 返回执行结果

      alt 结果成功
        Scheduler->>StateMgr: "updateStatus(success) (scheduler.ts:500)"
      else 结果取消
        Scheduler->>StateMgr: "updateStatus(cancelled) (scheduler.ts:502) "
      else 其他错误
        Scheduler->>StateMgr: "updateStatus(error) (scheduler.ts:504)"
      end
    end
  end

  Note over Scheduler: 处理请求队列中的下一批
  Scheduler->>Scheduler: "_processNextInRequestQueue() (scheduler.ts:508)"

  Scheduler->>StateMgr: 获取 completedBatch
  StateMgr-->>Caller: "返回 completedToolCalls[] (scheduler.ts:261)"
```

## Agent 执行详细序列

### 序列图 5: 本地Agent执行流程

```mermaid
sequenceDiagram
  participant Caller as 调用者
  participant AgentExec as LocalAgentExecutor.create
  participant ToolRegistry as 工具注册表
  participant GeminiChat as GeminiChat实例
  participant Agent as LocalAgentExecutor实例

  Caller->>AgentExec: "create(definition, runtimeContext, onActivity) (local-executor.ts:107)"

  Note over AgentExec,ToolRegistry: 第1步：创建独立工具注册表
  AgentExec->>ToolRegistry: "new ToolRegistry(runtimeContext) (local-executor.ts:113)"
  AgentExec->>ToolRegistry: 获取父工具注册表 (local-executor.ts:117)
  AgentExec->>AgentExec: 获取所有Agent名称 (local-executor.ts:118-120)

  Note over AgentExec,ToolRegistry: 第2步：注册工具
  alt 有 toolConfig
    loop 遍历 toolConfig.tools
      AgentExec->>ToolRegistry: 根据名称注册工具 (local-executor.ts:149-161)
    end
  else 无 toolConfig
    AgentExec->>ToolRegistry: 注册所有父工具 (local-executor.ts:164-167)
  end

  AgentExec->>ToolRegistry: sortTools() (local-executor.ts:169)

  Note over AgentExec,Agent: 第3步：创建Agent实例并运行
  AgentExec->>Agent: "new LocalAgentExecutor(...) (local-executor.ts:194-214)"
  Agent->>Agent: 设置 agentId (parentPrefix + name + random)

  Caller->>Agent: "run(inputs, signal) (local-executor.ts:403)"

  Note over Agent: 第4步：设置超时和初始化
  Agent->>Agent: 设置超时控制器 (local-executor.ts:409-414)
  Agent->>Agent: 合并 AbortSignal (local-executor.ts:417)
  Agent->>Agent: 记录 AgentStartEvent (local-executor.ts:419-422)

  Note over Agent,GeminiChat: 第5步：准备执行环境
  Agent->>Agent: 准备增强输入参数 (local-executor.ts:428-433)
  Agent->>Agent: prepareToolsList() (local-executor.ts:435)
  Agent->>GeminiChat: createChatObject(inputs, tools) (local-executor.ts:436)
  Agent->>Agent: 构建查询消息 (local-executor.ts:437-440)

  Note over Agent,GeminiChat: 第6步：Agent执行循环
  loop 主循环
    break 达到终止条件
      Agent->>Agent: 设置 terminateReason (local-executor.ts:446)
    end

    break 信号已中止
      Agent->>Agent: 设置 terminateReason (local-executor.ts:451-456)
    end

    Agent->>GeminiChat: executeTurn() (local-executor.ts:459-465)
    GeminiChat->>GeminiChat: callModel() (local-executor.ts:234-236)

    Note over Agent: 处理模型响应
    alt 信号已中止
      Agent->>Agent: 返回 stop 结果 (local-executor.ts:238-247)
    end

    alt 无函数调用 (协议违规)
      Agent->>Agent: 返回 stop + ERROR (local-executor.ts:250-260)
    end

    Agent->>Agent: processFunctionCalls(functionCalls) (local-executor.ts:262-263)

    alt 调用了 complete_task 工具
      Agent->>Agent: 返回 stop + GOAL (local-executor.ts:264-271)
    end

    alt 状态为 continue
      Agent->>Agent: 继续下一轮 (local-executor.ts:274-277)
    else 状态为 stop
      break 停止循环
        Agent->>Agent: 保存 terminateReason (local-executor.ts:468-472)
      end
    end
  end

  Note over Agent: 第7步：统一恢复逻辑
  alt "终止原因可恢复 (TIMEOUT/MAX_TURNS/ERROR_NO_COMPLETE_TASK)"
    Agent->>GeminiChat: executeFinalWarningTurn() (local-executor.ts:488-493)

    Note over Agent: 恢复轮执行（60秒宽限期）
    Agent->>GeminiChat: 发送警告消息 (local-executor.ts:336-339)
    Agent->>GeminiChat: executeTurn() (local-executor.ts:347-353)

    alt 恢复成功 (调用了 complete_task)
      Agent->>Agent: 设置 terminateReason = GOAL (local-executor.ts:359-364)
    else 恢复失败
      Agent->>Agent: 恢复失败处理 (local-executor.ts:368-372)
    end
  end

  Note over Agent: 第8步：返回最终结果
  alt 成功完成
    Agent->>Agent: 返回 result + terminate_reason (local-executor.ts:530-534)
  else 失败终止
    Agent->>Agent: 返回 error result (local-executor.ts:536-540)
  end

  Agent-->>Caller: 返回 OutputObject
```

## 交互模式UI启动序列

### 序列图 6: 交互式UI启动流程

```mermaid
sequenceDiagram
    participant Caller as 调用者(gemini.tsx)
    participant StartUI as startInteractiveUI(gemini.tsx:179)
    participant ConsolePatch as ConsolePatcher
    participant InkRender as Ink渲染器
    participant AppWrapper as AppWrapper组件
    participant AppContainer as AppContainer组件

    Caller->>StartUI: startInteractiveUI(config, settings, startupWarnings, workspaceRoot, resumedSessionData, initializationResult)

    StartUI->>StartUI: 确定是否使用备用缓冲区 (gemini.tsx:187-190)
    StartUI->>StartUI: 启用鼠标事件 (gemini.tsx:192-197)

    StartUI->>ConsolePatch: 创建控制台补丁 new ConsolePatcher() (gemini.tsx:202)
    StartUI->>ConsolePatch: 应用补丁 consolePatcher.patch() (gemini.tsx:208)

    StartUI->>StartUI: 创建工作标准IO createWorkingStdio() (gemini.tsx:211)

    StartUI->>AppWrapper: 创建包装组件 (gemini.tsx:214-245)
    AppWrapper->>AppContainer: 渲染AppContainer (gemini.tsx:231-237)

    StartUI->>InkRender: 渲染React应用 render() (gemini.tsx:247-272)
    InkRender-->>StartUI: 返回渲染实例

    StartUI->>StartUI: 检查更新 checkForUpdates() (gemini.tsx:274-283)

    StartUI->>StartUI: 注册清理函数 (gemini.tsx:285)

    StartUI-->>Caller: 返回void
```

## 关键文件路径参考

### 核心文件位置

| 文件       | 路径                                    | 主要功能                   |
| ---------- | --------------------------------------- | -------------------------- |
| CLI入口    | `packages/cli/index.ts`                 | CLI二进制入口，异常处理    |
| 主模块     | `packages/cli/src/gemini.tsx`           | 主应用逻辑，启动流程编排   |
| 参数解析   | `packages/cli/src/config/config.ts`     | CLI参数解析和验证          |
| 设置管理   | `packages/cli/src/config/settings.ts`   | 设置加载、合并、持久化     |
| 非交互模式 | `packages/cli/src/nonInteractiveCli.ts` | 非交互执行逻辑             |
| 应用初始化 | `packages/cli/src/core/initializer.ts`  | 应用启动初始化、认证、主题 |
| UI容器     | `packages/cli/src/ui/AppContainer.tsx`  | 主UI组件                   |

### 核心包文件位置

| 文件        | 路径                                           | 主要功能                       |
| ----------- | ---------------------------------------------- | ------------------------------ |
| Config类    | `packages/core/src/config/config.ts`           | 全局配置管理、工具/Agent注册表 |
| 工具注册表  | `packages/core/src/tools/tool-registry.ts`     | 工具注册、查找、排序           |
| 调度器      | `packages/core/src/scheduler/scheduler.ts`     | 工具调用调度、策略检查         |
| 状态管理器  | `packages/core/src/scheduler/state-manager.ts` | 调度器状态管理                 |
| 工具执行器  | `packages/core/src/scheduler/tool-executor.ts` | 工具实际执行                   |
| Agent执行器 | `packages/core/src/agents/local-executor.ts`   | 本地Agent执行、循环控制        |
| Agent注册表 | `packages/core/src/agents/registry.ts`         | Agent注册、查找                |
| Gemini聊天  | `packages/core/src/chat/gemini-chat.ts`        | Gemini API交互                 |

## 关键方法调用链

### 主流程调用链

1. **完整启动流程（包含沙箱处理）**

   ```
   index.ts:main() → gemini.tsx:main()
     → loadSettings()                           # 加载设置
     → parseArguments()                       # 解析参数
     → loadCliConfig() (第1次)              # 初步配置（用于认证刷新）
     → refreshAuth()                           # 刷新认证
     → start_sandbox() 或 relaunchAppInChildProcess()  # 沙箱或子进程
     → loadCliConfig() (第2次)              # 完整配置加载
     → initializeApp()                         # 应用初始化
     → startInteractiveUI() 或 runNonInteractive() # 进入UI或非交互模式
   ```

2. **配置加载链**

   ```
   loadCliConfig() (config.ts:416)
     → loadSettings()                          # 加载工作区设置
     → ExtensionManager 构造和 loadExtensions()  # 创建并加载扩展
     → loadServerHierarchicalMemory()            # 加载内存/上下文
     → resolveTelemetrySettings()                # 解析遥测设置
     → createPolicyEngineConfig()                # 创建策略引擎配置
     → loadSandboxConfig()                     # 加载沙箱配置
     → new Config()                            # 创建配置对象
   ```

3. **非交互模式调用链**

   ```
   runNonInteractive() (nonInteractiveCli.ts:58)
     → setupStdinCancellation()                 # 设置Ctrl+C监听
     → new Scheduler()                          # 创建调度器
     → getGeminiClient()                      # 获取Gemini客户端
     → handleSlashCommand() 或 handleAtCommand()  # 处理命令
     → sendMessageStream()                       # 发送消息流
     → Scheduler.schedule()                      # 调度工具调用
     → 处理流式响应循环 → 返回结果
   ```

4. **工具调度链**

   ```
   Scheduler.schedule() (scheduler.ts:141)
     → _startBatch()                          # 批处理启动
     → _validateAndCreateToolCall()              # 验证和创建工具调用
     → _processQueue()                         # 处理队列
     → _processNextItem()                      # 处理下一项
     → _processValidatingCall()                # 处理验证中的调用
     → _processToolCall()                      # 处理工具调用
       → checkPolicy()                          # 策略检查
       → resolveConfirmation()                    # 用户确认（如需要）
       → ToolExecutor.execute()                   # 执行工具
   ```

5. **Agent执行链**

   ```
   LocalAgentExecutor.create() (local-executor.ts:107)
     → new ToolRegistry(独立)                   # 创建独立工具注册表
     → 注册父工具到独立注册表
     → new LocalAgentExecutor()

   LocalAgentExecutor.run() (local-executor.ts:403)
     → createChatObject()                      # 创建聊天对象
     → 执行循环:
       → executeTurn()                         # 执行一轮
         → callModel()                          # 调用模型
         → processFunctionCalls()                 # 处理函数调用
         → scheduleAgentTools()                  # 调度Agent工具
       → 检查终止条件（超时、最大轮次）
       → 恢复逻辑（如需要）
   ```

## WebStorm 调试断点建议

### 启动流程断点

| 调试目标       | 文件路径                      | 行号 | 说明                   |
| -------------- | ----------------------------- | ---- | ---------------------- |
| main入口       | `packages/cli/src/gemini.tsx` | 294  | main() 函数入口        |
| 设置加载       | `packages/cli/src/gemini.tsx` | 313  | loadSettings()         |
| 参数解析       | `packages/cli/src/gemini.tsx` | 335  | parseArguments()       |
| 初步配置       | `packages/cli/src/gemini.tsx` | 385  | 第一次 loadCliConfig() |
| 沙箱检查       | `packages/cli/src/gemini.tsx` | 446  | 沙箱判断               |
| 完整配置       | `packages/cli/src/gemini.tsx` | 509  | 第二次 loadCliConfig() |
| 应用初始化     | `packages/cli/src/gemini.tsx` | 611  | initializeApp()        |
| 交互模式入口   | `packages/cli/src/gemini.tsx` | 658  | startInteractiveUI()   |
| 非交互模式入口 | `packages/cli/src/gemini.tsx` | 743  | runNonInteractive()    |

### 配置加载断点

| 调试目标     | 文件路径                            | 行号 | 说明                              |
| ------------ | ----------------------------------- | ---- | --------------------------------- |
| 配置加载入口 | `packages/cli/src/config/config.ts` | 416  | loadCliConfig()                   |
| 扩展管理创建 | `packages/cli/src/config/config.ts` | 465  | ExtensionManager 构造             |
| 扩展加载     | `packages/cli/src/config/config.ts` | 474  | extensionManager.loadExtensions() |
| 内存加载     | `packages/cli/src/config/config.ts` | 484  | loadServerHierarchicalMemory()    |
| 策略创建     | `packages/cli/src/config/config.ts` | 651  | createPolicyEngineConfig()        |
| Config构造   | `packages/cli/src/config/config.ts` | 685  | new Config()                      |

### 非交互模式断点

| 调试目标     | 文件路径                                 | 行号 | 说明                                |
| ------------ | ---------------------------------------- | ---- | ----------------------------------- |
| 非交互入口   | `packages/cli/src/nonInteractiveCli.ts`  | 58   | runNonInteractive()                 |
| 调度器创建   | `packages/cli/src/nonInteractiveCli.ts`  | 213  | new Scheduler()                     |
| 消息发送     | `packages`/cli/src/nonInteractiveCli.ts` | 300  | sendMessageStream()                 |
| 流式响应处理 | `packages/cli/src/nonInteractiveCli.ts`  | 310  | for await (event of responseStream) |
| 工具调用处理 | `packages/cli/src/nonInteractiveCli.ts`  | 334  | ToolCallRequest 事件                |
| 调度器调用   | `packages/cli/src/nonInteractiveCli.ts`  | 395  | scheduler.schedule()                |

### 调度器断点

| 调试目标   | 文件路径                                   | 行号 | 说明                          |
| ---------- | ------------------------------------------ | ---- | ----------------------------- |
| 调度入口   | `packages/core/src/scheduler/scheduler.ts` | 141  | schedule()                    |
| 批处理启动 | `packages/core/src/scheduler/scheduler.ts` | 231  | \_startBatch()                |
| 工具验证   | `packages/core/src/scheduler/scheduler.ts` | 287  | \_validateAndCreateToolCall() |
| 队列处理   | `packages/core/src/scheduler/scheduler.ts` | 328  | \_processQueue()              |
| 策略检查   | `packages/core/src/scheduler/scheduler.ts` | 407  | checkPolicy()                 |
| 工具执行   | `packages/core/src/scheduler/scheduler.ts` | 470  | \_execute()                   |

### Agent执行断点

| 调试目标  | 文件路径                                     | 行号 | 说明                |
| --------- | -------------------------------------------- | ---- | ------------------- |
| Agent创建 | `packages/core/src/agents/local-executor.ts` | 107  | create()            |
| 工具注册  | `packages/core/src/agents/local-executor.ts` | 149  | registerToolByName  |
| Agent运行 | `packages/core/src/agents/local-executor.ts` | 403  | run()               |
| 执行循环  | `packages/core/src/agents/local-executor.ts` | 442  | while (true) 主循环 |
| 一轮执行  | `packages/core/src/agents/local-executor.ts` | 223  | executeTurn()       |
| 模型调用  | `packages/core/src/agents/local-executor.ts` | 234  | callModel()         |

## 总结

Gemini
CLI 的架构设计体现了清晰的关注点分离和模块化设计。通过上述序列图可以清楚地看到：

1. **启动阶段**：参数解析、配置加载、认证刷新、沙箱检查
2. **配置阶段**：扩展加载、内存加载、策略创建、配置对象构建
3. **执行阶段**：根据模式选择交互或非交互路径
4. **工具调用**：通过调度器协调工具执行、策略检查、用户确认
5. **Agent系统**：独立的执行环境、工具注册、循环控制、恢复机制
6. **UI系统**：React-based终端界面、提供丰富的交互体验

每个模块都有明确的职责边界，通过标准接口进行通信，使得系统易于维护、测试和扩展。
