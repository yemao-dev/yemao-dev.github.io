<div align="center">
<h3>KingFlow · 国内直连 AI API 中转</h3>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/端点-/v1-1E90FF" alt="Endpoint"></a>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/模型-Claude%20%7C%20GPT-7B68EE" alt="Models"></a>
</div>

# Claude Code 接入国内中转 API 实战：倍率透明、后台可查

用 Claude Code 写了几个月代码，最大的体会不是它有多聪明，而是 token 烧得是真的快。每天开两三个 session，让它读项目、改逻辑、跑测试，月底一看账单经常被吓一跳。后来折腾出一套自己用着舒服的方案，端点换成 KingFlow（https://www.kingflow.ai），把成本压下来还顺手把账算清楚了。这篇就讲怎么接、怎么配、怎么对账，全是实操。

## 一、官方 API 跑 Claude Code，痛在哪

先说为什么不直接用官方。不是说官方不好，是几个现实问题：

**太贵。** Claude Code 是个吃 token 的猛兽。它干活前会把相关文件、目录结构、历史对话一股脑塞进上下文，一次复杂任务输入轻松上十万 token。旗舰模型按官方价跑，一天下来几美元很正常，重度使用一个月上百刀也不稀奇。

**token 烧得快还不知不觉。** 你以为只是让它改个函数，它默默读了二十个文件做上下文。等你反应过来，额度已经掉了一大截。Claude Code 默认不在终端实时滚动显示「这一步花了多少钱」，烧钱过程基本是黑盒。

**账单不好算。** 官方账单是按整体用量出的，你很难拆清楚到底是哪个项目、哪个时段、哪种模型花的钱。团队几个人共用一个 key 时更乱，谁也说不清这个月为什么超了。

这三个问题叠加，结果就是「能用，但心里没底」。我要的是用着省心、花钱明白。

## 二、KingFlow 的解法：倍率透明 + 国内直连

KingFlow 是个 API 中转，核心解决两件事——**价格看得见，连接连得上**。

**倍率透明，后台全程可查。** KingFlow 按倍率计费，GPT 系 0.1 倍率起、Claude 0.6 倍率，具体价格登录后台一眼能看到。更关键的是后台把每一次请求都记下来：调用日志、剩余余额、每次消耗的 token 用量，全都列得清清楚楚。我现在每天收工前扫一眼日志，今天哪个 session 是大头、哪条命令意外吃了几万 token，一目了然。这点比官方那种「月底给你一个总数」舒服太多。

**国内直连，不用折腾梯子。** 端点是 https://www.kingflow.ai/v1，国内网络直接连得上，不用为了让 Claude Code 跑起来在终端里挂一堆代理。延迟稳定，跑长任务不容易中途断。

对比一下就清楚了：

| 维度 | 官方 API 直连 | KingFlow 中转 |
| --- | --- | --- |
| 国内访问 | 需自备稳定网络环境 | 直连 https://www.kingflow.ai/v1 |
| 计费方式 | 官方原价 | 倍率计费，Claude 0.6 / GPT 0.1 起 |
| token 用量可见 | 黑盒，月底汇总 | 后台逐条日志，实时可查 |
| 余额查询 | 较粗略 | 后台实时余额 |
| 上手门槛 | 改环境变量 | 同样改两个环境变量 |
| 先试后买 | 无 | 注册送额度，可先测 |

## 三、Claude Code 配置实战：跑通第一条命令

接入只动两个环境变量，不用改 Claude Code 任何源码。

Claude Code 认 `ANTHROPIC_BASE_URL` 和 `ANTHROPIC_AUTH_TOKEN` 这两个变量。前者指向 KingFlow 端点，后者填你在 KingFlow 后台拿到的 key。

macOS / Linux，写进 `~/.zshrc` 或 `~/.bashrc`：

```bash
export ANTHROPIC_BASE_URL="https://www.kingflow.ai/v1"
export ANTHROPIC_AUTH_TOKEN="你的-KingFlow-key"
```

存盘后 `source ~/.zshrc` 让它生效。Windows 用 PowerShell：

```powershell
setx ANTHROPIC_BASE_URL "https://www.kingflow.ai/v1"
setx ANTHROPIC_AUTH_TOKEN "你的-KingFlow-key"
```

设完重开一个终端，让变量加载进来。然后跑通第一条命令验证：

```bash
cd 你的项目目录
claude "用一句话总结这个项目是做什么的"
```

正常的话，Claude Code 会读项目、回一句总结。这时切回 KingFlow 后台看日志，应该刚好多出一条调用记录，带着这次的 token 用量和扣费。能对上，就说明链路完全通了。

懒得改全局环境变量的话，也可以临时只在当前 session 生效：

```bash
ANTHROPIC_BASE_URL="https://www.kingflow.ai/v1" \
ANTHROPIC_AUTH_TOKEN="你的-KingFlow-key" \
claude "帮我重构 utils 里的日期处理函数"
```

想用 cURL 单独验端点活没活，也行：

```bash
curl https://www.kingflow.ai/v1/messages \
  -H "x-api-key: 你的-KingFlow-key" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-haiku-4-5",
    "max_tokens": 64,
    "messages": [{"role": "user", "content": "ping"}]
  }'
```

返回正常 JSON，端点就是通的。

## 四、成本可控：先测后充，后台对账

新号注册一般送测试额度，强烈建议别一上来就充钱。先用送的额度跑几次你**真实的日常任务**——读个中型项目、改一段复杂逻辑、让它跑一轮测试。把速度、稳定性、扣费都摸一遍，价格和体验都满意了再充。

充值后我的对账习惯就两步：每次干完一段活，回后台看日志；每周做一次小复盘，看这周钱主要花在什么类型的任务上。几次下来你会很清楚哪些操作性价比高、哪些纯属浪费 token，慢慢就把用法优化出来了。这种「花钱明白」的感觉，是黑盒账单给不了的。

## 五、模型怎么选：贵的留给难活

KingFlow 上 Claude 几个型号都在，关键是按任务匹配，别一律上最贵的：

- **claude-opus-4-8（旗舰）**——写复杂逻辑、大规模重构、架构设计、啃陌生代码库这种真需要脑子的活，用它。贵但值，关键时刻能省下你大量来回调试的时间。
- **claude-sonnet-4-6（均衡）**——日常写功能、改中等复杂度的代码，能力和成本平衡得不错，可以当默认主力。
- **claude-haiku-4-5（高频/低成本）**——跑高频小任务首选：批量改命名、生成样板代码、写简单单测、格式化整理、快速问答。便宜量大，闭眼用。

我自己的搭配是 haiku 兜底高频琐事、sonnet 干主力、opus 专啃硬骨头。Claude Code 里随时可以切模型，养成习惯后单位时间产出的「每块钱价值」明显高一截。

## 六、生产业务请留备用方案

最后一句要紧的话：**中转适合个人开发、日常写码、跑实验，生产业务千万自己留备用方案。**

任何第三方中转都有它的依赖链，网络、上游、额度任何一环出问题都可能影响调用。个人折腾代码无所谓，但如果你把它接进了线上服务的关键路径，一定要有降级策略——比如保留官方直连作为 fallback、设好超时和重试、对关键调用做监控告警。把它当成「省钱提效的好工具」，而不是「唯一不能倒的依赖」，心态和架构就都对了。

## 七、FAQ

**Q1：接入 KingFlow 需要改 Claude Code 的代码吗？**
不用。只配 `ANTHROPIC_BASE_URL=https://www.kingflow.ai/v1` 和 `ANTHROPIC_AUTH_TOKEN` 两个环境变量，Claude Code 本身一行不动。

**Q2：怎么知道每次到底花了多少 token？**
登录 KingFlow 后台看日志，每一次调用都有独立记录，带 token 用量和扣费，余额也实时显示。这正是它相比官方黑盒账单的优势。

**Q3：倍率具体怎么算，价格在哪看？**
按倍率计费，Claude 0.6 倍率、GPT 0.1 倍率起，最终单价以后台展示为准。建议先用注册送的额度跑几次真实任务，自己算一笔再决定充多少。

**Q4：国内网络能直接跑通吗？**
端点 https://www.kingflow.ai/v1 面向国内直连，正常网络下不用额外挂代理就能让 Claude Code 跑起来。若首条命令没通，先检查环境变量是否拼对、终端是否重开加载了变量。
