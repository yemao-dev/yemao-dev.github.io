# OpenAI Codex 安装配置 KingFlow 超详细教程 — AI 编程工具 Codex 实战配置与常见错误总结

<div align="center">

[![官网](https://img.shields.io/badge/官网-www.kingflow.ai-2563eb?style=for-the-badge)](https://www.kingflow.ai)
[![Codex](https://img.shields.io/badge/支持-OpenAI%20Codex-16a34a?style=for-the-badge)](https://www.kingflow.ai)

</div>

在现代软件开发中，**代码规模越来越大、技术栈越来越复杂**，开发者花在"理解代码、改动代码、反复验证"的时间，往往远多于真正写新功能的时间。为了解决这一问题，**Codex** 应运而生。

**Codex（Codex CLI）** 是一个运行在终端中的 AI 编程助手。与普通聊天式 AI 不同，它可以直接读取你的项目代码、理解文件结构，在你的确认下修改源码、执行命令，并一步步完成真实的开发任务。你可以把它理解为一个"懂代码、会动手、但始终受你控制的编程搭档"。

本文将带你从零开始学习 Codex 的**安装、配置与实际使用方法**，包括如何接入 **KingFlow** 第三方 API、如何在终端和 VS Code 中高效使用，以及遇到常见问题时如何快速排查。

***

## 安装前准备（所有系统通用）

* **Node.js 22+**
* **npm 10+**
* 稳定网络连接

> Windows 额外注意：OpenAI 官方提到 Windows 支持偏"实验性"，更稳的方式是用 WSL 环境。

***

## 安装 Codex CLI

### Windows

1. 安装 **Git Bash**（按安装向导一直下一步即可）。
2. 安装 Node.js（建议装最新 LTS）。
3. 安装 Codex CLI（在 CMD / PowerShell 里执行）：

```bash
npm install -g @openai/codex
```

4. 验证：

```bash
codex --version
```

### macOS

```bash
npm install -g @openai/codex
codex --version
```

必要时加 sudo。OpenAI 官方也提供 Homebrew 安装方式（可选）：`brew install codex`。

### Linux

```bash
sudo npm install -g @openai/codex
codex --version
```

***

## 配置 KingFlow 作为第三方 API

Codex CLI 会读取你的配置文件：一般在 `~/.codex/`（Windows 也是用户目录下的 `.codex`）。需创建两份文件：

* `auth.json`：放密钥
* `config.toml`：放模型与网关配置

### Windows 配置路径与文件

1. 进入用户目录的 `.codex`（示例：`C:\Users\testuser\.codex`）。如果看不到，先在资源管理器开启"显示隐藏项目"。
2. 没有就手动创建 `.codex` 文件夹，并在其中创建 `auth.json` 与 `config.toml`。

**auth.json（把 sk-xxx 换成你的 KingFlow API Key）**

```json
{"OPENAI_API_KEY": "sk-xxx"}
```

**config.toml**

```toml
model_provider = "kingflow"
model = "gpt-5-codex"
model_reasoning_effort = "high"
disable_response_storage = true
preferred_auth_method = "apikey"

[model_providers.kingflow]
name = "kingflow"
base_url = "https://www.kingflow.ai/v1"
wire_api = "responses"
```

### macOS / Linux 配置命令

```bash
mkdir -p ~/.codex
touch ~/.codex/auth.json
touch ~/.codex/config.toml
```

编辑 `auth.json`（粘贴同样的 JSON）与 `config.toml`。**务必保证上下一致**：`model_provider = "kingflow"` 要和 `[model_providers.kingflow]` 的段名一致。

### 配置改完一定要"重启终端"

**关闭终端 / 重启终端后再启动 codex**，让配置生效。

***

## 启动与基本使用（终端）

进入你的项目目录：

```bash
cd your-project-folder
codex
```

你也可以直接在命令后跟一个初始任务，例如让它先解释仓库结构：

```bash
codex "Explain this codebase to me"
```

### 推荐的使用习惯（很实用）

* **先让它"读项目、给计划"**：比如"先扫描项目结构，列出你会修改哪些文件，再开始动手"。
* **小步提交**：每次只让它做一件事（修一个 bug / 加一个功能点）。
* **用 Git 做检查点**：Codex 会改文件，任务前后做 git checkpoint，方便回滚。

***

## 交互技巧：Slash 命令与快捷操作

在 Codex 交互界面里，输入 `/` 可以打开 slash 命令菜单，用来切换模型、调整权限、总结对话等。常见命令：

* `/status`：查看当前会话配置 / 状态
* `/new`：开新会话（清空上下文）
* `/model`：切换模型
* `/init`：初始化模板 / 设置（视版本而定）

很多版本支持用 `!` 直接跑终端命令（例如 `!git status` / `!ls`），能减少"让模型代跑命令"的成本。

***

## VS Code 插件 Codex

完成上述 `.codex` 配置后，在 VS Code 扩展商店搜索并安装 **codex**，安装后会出现在侧边栏（有时在折叠区）。配置文件与 CLI 共用，无需重复填写。

***

## 常见问题（FAQ）与排查清单

### Q1：`codex: command not found` / 找不到命令

常见原因是 npm 全局安装路径没加入 PATH，或安装没成功。

* 先运行 `codex --version` 验证是否安装成功。
* 重新安装：`npm install -g @openai/codex`。

### Q2：Linux/macOS 安装时报权限错误（EACCES）

用 `sudo npm install -g @openai/codex`。（更长期的做法是把 npm 全局目录改到用户目录，属于通用 Node/npm 运维。）

### Q3：Windows 找不到 `.codex` 文件夹

需要在资源管理器里打开"显示隐藏的项目"，因为 `.codex` 是隐藏目录。

### Q4：配置了 Key 但仍提示未认证 / 401

确认两点：

1. `auth.json` 内容必须是：`{"OPENAI_API_KEY": "sk-xxx"}`
2. **重启终端**后再运行 codex。

### Q5：一直连不上 / 超时 / 网络错误

* 检查 `base_url` 是否完全一致（`https://www.kingflow.ai/v1`）。
* 公司 / 校园网络可能需要放行相关域名（这是网络环境问题，不是 Codex 本身）。

### Q6：模型不可用 / 报 model not found

先用提供的模型名（如 `gpt-5-codex`）。如果你在 KingFlow 模型列表看到的名称不同，就以实际可用模型为准（模型名不匹配会直接失败）。

### Q7：`config.toml` 写了但好像没生效

最常见原因：

* `model_provider = "X"` 和 `[model_providers.X]` **名字不一致**（务必都写 `kingflow`）。
* 忘记重启终端。

### Q8：怎么升级 / 更新 Codex CLI？

```bash
npm i -g @openai/codex@latest
```

### Q9：有哪些命令行参数 / 高级配置可以查？

* CLI 默认从 `~/.codex/config.toml` 读取配置，也支持用 `-c key=value` 临时覆盖。
* 配置字段的完整参考见 OpenAI 官方 config reference 页面。

***

> 🚀 用 KingFlow 接入 Codex，国内直连、低延迟、按量计费：**https://www.kingflow.ai**
