# start.js 与启动流程对照 + 断点建议

根据你采用的方案二，`start.js`
是调试的真正入口。让我对照代码和启动流程图给出断点建议。

---

## start.js 执行流程

| 阶段             | 行号  | 说明                           | 对应流程图          |
| ---------------- | ----- | ------------------------------ | ------------------- |
| **检查构建状态** | 28-33 | `check-build-status.js`        | 未在图中（辅助）    |
| **获取沙箱配置** | 35-45 | `sandbox_command.js`           | 未在图中（辅助）    |
| **设置调试参数** | 49-58 | `DEBUG` 环境变量检查判断       | 未在图中            |
| **设置环境变量** | 61-67 | `CLI_VERSION`, `DEV=true`      | 未在图中            |
| **设置调试标志** | 69-72 | `GEMINI_CLI_NO_RELAUNCH=true`  | 未在图中            |
| **启动子进程**   | 74    | `spawn('node', nodeArgs, env)` | 启动 `packages/cli` |

**子进程执行链**：

```
start.js → packages/cli/index.ts → packages/cli/src/gemini.tsx:main()
```

---

## 推荐的断点设置（按执行顺序）

### 阶段 1：在 start.js 中设置断点（可选，用于理解启动逻辑）

| 文件路径           | 行号 | 说明                                               | 建议优先级                |
| ------------------ | ---- | -------------------------------------------------- | ------------------------- |
| `scripts/start.js` | 28   | `execSync('node ./scripts/check-build-status.js')` | 低（可以跳过）            |
| `scripts/start.js` | 35   | `execSync('node scripts/sandbox_command.js')`      | 低（可以跳过）            |
| `scripts/start.js` | 49   | `DEBUG` 环境变量检查判断                           | 低                        |
| `scripts/start.js` | 54   | `nodeArgs.push('--inspect-brk')`                   | **高** - 调试调试参数设置 |
| `scripts/start.js` | 56   | `nodeArgs.push(join(root, 'packages', 'cli'))`     | 中 - 确认子进程路径       |
| `scripts/start.js` | 74   | `spawn('node', nodeArgs, env)`                     | 高 - 观察启动配置         |

**注意**：在 `start.js`
中设置断点后，程序会在子进程中运行，你需要在子进程的代码中打断点。继续向下看子进程中的断点。

---

### 阶段 2：在 packages/cli/index.ts 中设置断点（全局异常处理）

| 文件路径                | 行号 | 说明                           | 建议优先级                            |
| ----------------------- | ---- | ------------------------------ | ------------------------------------- |
| `packages/cli/index.ts` | 17   | `uncaughtException` 监听器注册 | 低                                    |
| `packages/cli/index.ts` | 38   | `main()` 调用入口              | **高** - 这是进入子进程后的第一个断点 |

**说明**：这是"胶水层"，业务逻辑在 `gemini.tsx` 中。

---

### 阶段 3：在 packages/cli/src/gemini.tsx 中按流程顺序设置断点

#### 3.1 初始化和设置加载

| 文件路径                      | 行号 | 说明                               | 对应流程图        |
| ----------------------------- | ---- | ---------------------------------- | ----------------- |
| `packages/cli/src/gemini.tsx` | 294  | `main()` 函数开始                  | **最高优先级**    |
| `packages/cli/src/gemini.tsx` | 304  | `patchStdio()`                     | 设置 IO 补丁      |
| `packages/cli/src/gemini.tsx` | 311  | `setupUnhandledRejectionHandler()` | 异常处理          |
| `packages/cli/src/gemini.tsx` | 313  | `loadSettings()`                   | **高** - 加载设置 |
| `packages/cli/src/gemini.tsx` | 321  | `loadTrustedFolders()`             | 加载信任文件夹    |
| `packages/cli/src/gemini.tsx` | 329  | `cleanupCheckpoints()`             | 清理过期资源      |
| `packages/cli/src/gemini.tsx` | 335  | `parseArguments()`                 | **高** - 解析参数 |
| `packages/cli/src/gemini.tsx` | 385  | 第一次 `loadCliConfig()`           | **高** - 初步配置 |
| `packages/cli/src/gemini.tsx` | 394  | `validateAuthMethod()`             | 验证认证方法      |
| `packages/cli/src/gemini.tsx` | 407  | `refreshAuth()`                    | 刷新认证          |
| `packages/cli/src/gemini.tsx` | 436  | `loadRemoteAdminSettings()`        | 加载远程管理设置  |
| `packages/cli/src/gemini.tsx` | 443  | `runDeferredCommand()`             | 执行延迟命令      |

#### 3.2 沙箱检查和子进程启动

| 文件路径                      | 行号 | 说明                          | 对应流程图                    |
| ----------------------------- | ---- | ----------------------------- | ----------------------------- |
| `packages/cli/src/gemini.tsx` | 446  | 沙箱判断                      | 中                            |
| `packages/cli/src/gemini.tsx` | 450  | `loadSandboxConfig()`         | 中                            |
| `packages/cli/src/gemini.tsx` | 493  | `start_sandbox()`             | **高** - 启动沙箱（如果启用） |
| `packages/cli/src/gemini.tsx` | 500  | `relaunchAppInChildProcess()` | **高** - 重新启动（无沙箱）   |

**注意**：如果启用沙箱，程序会在这里退出并重启。你需要：

1. 在 `start_sandbox()` 中打断点
2. 或者在新的子进程中继续打断点

#### 3.3 最终配置加载和应用初始化

| 文件路径                      | 行号 | 说明                               | 对应流程图            |
| ----------------------------- | ---- | ---------------------------------- | --------------------- | --- |
| `packages/cli/src/gemini.tsx` | 509  | 第二次 `loadCliConfig()`           | **高** - 完整配置加载 |
| `packages/cli/src/gemini.tsx` | 526  | 注册遥测和策略                     | 中                    |
| `packages/cli/src/gemini.tsx` | 543  | `--list-extensions` 处理           | 低                    |
| `packages/cli/src/gemini.tsx` | 553  | `--list-sessions` 处理             | 低                    |
| `packages/cli/src/gemini.tsx` | 574  | `--delete-session` 处理            | 低                    |
| `packages/cli/src/gemini.tsx` | 582  | 设置 stdin Raw Mode                | 中                    |
| `packages/cli/src/gemini.tsx` | 608  | `setupTerminalAndTheme()`          | 中                    |
| `packages/cli/src/gemini.tsx` | 611  | `initializeApp()`                  | **高** - 应用初始化   |
| `packages/cli/src/gemini`     | 40   | `performInitialAuth()`             | 中                    |
| `packages/cli/src/gemini`     | 45   | `validateTheme()`                  | 中                    |
| `packages/cli/src/gemini.tsx` | 624  | `runZedIntegration()`              | Zed 模式              | 低  |
| `packages/cli/src/gemini.tsx` | 636  | `SessionSelector.resolveSession()` | 处理 --resume         | 低  |

#### 3.4 交互模式路径

| 文件路径                      | 行号 | 说明                           | 对应流程图              |
| ----------------------------- | ---- | ------------------------------ | ----------------------- | --- |
| `packages/cli/src/gemini.tsx` | 658  | `startInteractiveUI()`         | **最高** - 交互模式入口 |
| `packages/cli/src/gemini.tsx` | 179  | `shouldEnterAlternateScreen()` | 判断备用缓冲区          | 低  |
| `packages/cli/src/gemini.tsx` | 202  | `consolePatcher.patch()`       | 中                      |

#### 3.5 非交互模式路径

| 文件路径                      | 行号 | 说明                                 | 对应流程图                |
| ----------------------------- | ---- | ------------------------------------ | ------------------------- | --- |
| `packages/cli/src/gemini.tsx` | 669  | `config.initialize()`                | 中                        |
| `packages/cli/src/gemini.tsx` | 674  | 读取 stdin 数据                      | 中                        |
| `packages/cli/src/gemini.tsx` | 690  | `hookSystem.fireSessionStartEvent()` | SessionStart 钩子         | 低  |
| `packages/cli/src/gemini.tsx` | 743  | `runNonInteractive()`                | **最高** - 非交互模式入口 |

---

## 推荐的断点优先级

### 优先级 1（必设）：程序入口点

```
packages/cli/index.ts:38   // main() 调用 - 进入子进程后的第一个断点
packages/cli/src/gemini.tsx:294   // main() 函数开始 - 核心入口
```

### 优先级 2（关键）：配置加载和初始化

```
packages/cli/src/gemini.tsx:313   // loadSettings() - 设置加载
packages/cli/src/gemini.tsx:335   // parseArguments() - 参数解析
packages/cli/src/gemini.tsx:385   // 第一次 loadCliConfig() - 初步配置
packages/cli/src/gemini.tsx:407   // refreshAuth() - 认证刷新
packages/cli/src/gemini.tsx:611   // initializeApp() - 应用初始化
packages/cli/src/gemini.tsx:509   // 第二次 loadCliConfig() - 完整配置
packages/cli/src/gemini.tsx:658   // startInteractiveUI() - 交互模式入口
packages/cli/src/gemini.tsx:743   // runNonInteractive() - 非交互模式入口
```

### 优先级 3（按需）：功能特定断点

根据你要调试的功能，选择对应断点：

**调试工具调用**：

```
packages/cli/src/nonInteractiveCli.ts:58      // runNonInteractive()
packages/cli/src/nonInteractiveCli.ts:213     // new Scheduler()
packages/cli/src/nonInteractiveCli.ts:300     // sendMessageStream()
packages/cli/src/nonInteractiveCli.ts:310     // for await (event of responseStream)
packages/cli/src/nonInteractiveCli.ts:334     // ToolCallRequest 事件处理
packages/cli/src/nonInteractiveCli.ts:395     // scheduler.schedule()
```

**调试调度器**：

```
packages/core/src/scheduler/scheduler.ts:141     // schedule()
packages/core/src/scheduler/scheduler.ts:231        // _startBatch()
packages/core/src/scheduler/scheduler.ts:287       // _validateAndCreateToolCall()
packages/core/src/scheduler/scheduler.ts:328       // _processQueue()
packages/core/src/scheduler/scheduler.ts:407      // checkPolicy()
packages/core/src/scheduler/scheduler.ts:470       // _execute()
```

**调试 Agent 执行**：

```
packages/core/src/agents/local-executor.ts:107     // create()
packages/core/src/agents/local-executor.ts:403     // run()
packages/core/src/agents/local-executor.ts:442     // while (true) 主循环
packages/core/src/agents/local-executor.ts:223     // executeTurn()
packages/core/src/agents/local-executor.ts:234     // callModel()
```

---

## 快速开始调试步骤

1. 在 WebStorm 中按 `Ctrl+Shift+A` 打开"Run > Debug..."对话框

2. 选择或创建配置：
   - **Name**: `Gemini CLI Debug`
   - **Type**: `Node.js`
   - **Node parameters**:
     `--experimental-loader ts-node/esm/register.js --import ./scripts/start.js`

3. Working directory: `/Users/liujie/myprojects/gemini-cli`

4. 先在以下位置设置断点（按需）：
   - `packages/cli/src/gemini.tsx:294` (main 入口)
   - `packages/cli/src/gemini.tsx:313` (loadSettings)
   - `packages/cli/src/gemini.tsx:335` (parseArguments)
   - `packages/cli/src/gemini.tsx:385` (第一次 loadCliConfig)
   - `packages/cli/src/gemini.tsx:509` (第二次 loadCliConfig)
   - `packages/cli/src/gemini.tsx:611` (initializeApp)

5. 按 `Shift+F9` 启动调试

---

## 调试提示

- **断点不会在 start.js 中暂停**：`start.js`
  只启动子进程，真正的断点在子进程的代码中
- **子进程连接**：WebStorm 应该自动检测到子进程 Node.js 调试器
- **如果断点没停**：检查 WebStorm 的 Node.js 调试配置是否启用了源映射（Source
  Map）
- **沙箱场景**：如果启用沙箱，程序会在 `gemini.tsx:493`
  处退出并重启，需要在新的子进程中打断点
