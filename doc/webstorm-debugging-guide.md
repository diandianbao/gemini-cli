# WebStorm 调试 gemini-cli 指南

## 项目概述

gemini-cli 是一个基于 TypeScript 的 monorepo 项目，使用 pnpm/npm
workspaces 管理。

### 项目结构

```
gemini-cli/
├── packages/
│   ├── cli/          # CLI 主包 (入口: index.ts)
│   ├── core/         # 核心逻辑包
│   ├── a2a-server/   # A2A 服务器包
│   └── test-utils/   # 测试工具包
├── scripts/
│   └── start.js      # 开发启动脚本
└── package.json      # 根 package.json
```

### 技术栈

- **语言**: TypeScript (ES2022)
- **Node版本**: >=20.0.0
- **包管理器**: npm/pnpm
- **构建工具**: esbuild
- **调试器**: Node.js Inspector

---

## 一、环境准备

### 1.1 安装依赖

```bash
# 在项目根目录执行
npm install
或
npm install --registry=https://registry.npmjs.org/
# 或
pnpm install
```

### 1.2 编译项目

```bash
# 编译所有包
npm run build

# 或只编译需要的包
npm run build --workspace @google/gemini-cli-core
npm run build --workspace @
```

---

## 二、配置 WebStorm 调试器

### 2.1 方法一：使用 npm 脚本（推荐）

1. 在 WebStorm 中打开项目
2. 点击右上角的 "Add Configuration..." 或 `Run > Edit Configurations...`
3. 点击左上角的 `+` 按钮，选择 `npm`
4. 配置如下：

```
名称: Gemini CLI Debug
Package manager: npm
Scripts: debug
```

5. 点击 `OK` 保存

### 2.2 方法二：使用 Node.js 调试配置

1. 点击 `Run > Edit Configurations...`
2. 点击 `+` 按钮，选择 `Node.js`
3. 配置如下：

```
名称: Gemini CLI Direct Debug
Node interpreter: 使用项目中的 Node.js (建议使用 nvm 管理)
Node parameters: --inspect-brk scripts/start.js
Working directory: /Users/liujie/myprojects/gemini-cli
Environment variables:
  - DEBUG=1
  - DEV=true
```

### 2.3 方法三：调试已编译的代码

```bash
# 先编译项目
npm run build

# 然后配置 Node.js 调试器
Node parameters: --inspect-brk packages/cli/dist/index.js
Working
 directory: /Users/liujie/myprojects/gemini-cli
```

---

## 三、调试操作步骤

### 3.1 设置断点

1. 在 `packages/cli/src/` 目录下的源代码文件中设置断点
2. 或在 `packages/core/src/` 核心代码中设置断点

### 3.2 启动调试

1. 选择配置好的调试配置
2. 点击调试按钮（绿色虫子图标）或按 `Shift+F9`
3. 调试器会启动并在断点处暂停

### 3.3 传递命令行参数

在调试配置中添加 `Program arguments`：

```
# 示例：调试某个命令
help

# 示例：带参数的命令
analyze /path/to/file
```

---

## 四、调试不同场景

### 4.1 调试主命令流程

- **入口文件**: `packages/cli/index.ts:9` -> `main()`
- **主要流程**:
  1. `packages/cli/src/gemini.js` - 主程序入口
  2. `packages/core/src/agent/` - Agent 核心逻辑
  3. `packages/core/src/tools/` - 工具实现

### 4.2 调试 A2A 服务器

创建 Node.js 调试配置：

```
名称: Debug A2A Server
Node parameters: --inspect-brk packages/a2a-server/src/index.ts
Working directory: /Users/liujie/myprojects/gemini-cli
Environment variables:
  - PORT=41242
```

### 4.3 调试测试文件

创建 Vitest 调试配置：

```
名称: Debug Vitest Tests
Type: Vitest
File: 需要调试的测试文件路径
Working directory: /Users/liujie/myprojects/gemini-cli
```

---

## 五、环境变量配置

| 变量                     | 说明                          | 默认值  |
| ------------------------ | ----------------------------- | ------- |
| `DEBUG`                  | 启用调试模式                  | `0`     |
| `DEV`                    | 开发模式标识                  | `false` |
| `GEMINI_SANDBOX`         | 沙箱模式 (none/docker/podman) | -       |
| `NO_COLOR`               | 禁用彩色输出                  | -       |
| `GEMINI_CLI_NO_RELAUNCH` | 禁止进程重新启动              | -       |

---

## 六、常见问题

### 6.1 SourceMap 不生效

确保 `tsconfig.json` 中已启用 sourceMap：

```json
{
  "compilerOptions": {
    "sourceMap": true
  }
}
```

### 6.2 找不到模块

确保已编译项目或使用开发模式启动：

```bash
# 使用开发模式（自动编译）
npm run start
```

### 6.3 调试器无法连接

检查端口是否被占用，或指定不同的调试端口：

```
Node parameters: --inspect-brk=localhost:9230 scripts/start.js
```

### 6.4 TypeScript 类型检查

在 WebStorm 中启用实时类型检查：

1. `Preferences > Languages & Frameworks > TypeScript`
2. 勾选 `Recompile on changes`

---

## 七、快捷键参考

| 快捷键       | 功能                         |
| ------------ | ---------------------------- |
| `F8`         | Step Over (单步跳过)         |
| `F7`         | Step Into (单步进入)         |
| `Shift+F8`   | Step Out (跳出)              |
| `F9`         | Resume (继续执行)            |
| `Command+F8` | Toggle Breakpoint (切换断点) |
| `Shift+F9`   | Debug (启动调试)             |

---

## 八、调试技巧

### 8.1 条件断点

在断点上右键 > `More` 或 `Command+F8`，设置条件：

```javascript
// 只在特定条件下暂停
file === 'important.js' && line > 100;
```

### 8.2 日志断点

使用日志断点（不暂停，只输出表达式）：

```javascript
// 在断点设置中输入表达式
`User: ${user.name}, ID: ${user.id}`;
```

### 8.3 表达式求值

调试时在 Debug 面板中实时计算表达式：

```javascript
// 在 Evaluate Expression 窗口中
context.getToolNames();
```

---

## 九、参考资源

- [gemini-cli GitHub](https://github.com/google-gemini/gemini-cli)
- [WebStorm 调试文档](https://www.jetbrains.com/help/webstorm/debugging-javascript-in-webstorm.html)
- [Node.js Inspector](https://nodejs.org/en/docs/guides/debugging-getting-started/)
