<div align="center">

<h3>KingFlow · 国内直连 AI API 中转</h3>

<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
<a href="https://www.kingflow.ai/v1"><img src="https://img.shields.io/badge/兼容端点-%2Fv1-2E86DE" alt="KingFlow v1"></a>

</div>

# Codex 接入 KingFlow 兼容 API 桥接实战：本地代理配置全流程

如果你直接把 Codex 指向一个 OpenAI 兼容的中转地址，很可能第一步就卡住——Codex 客户端默认走的是 Responses API，而 KingFlow 对外暴露的是 OpenAI 兼容的 Chat Completions 形态。两者字段、路径、流式返回都不完全对齐。本文是我自己踩完坑之后整理的实战流程：用一层本地桥接把两种协议接上，让 Codex 稳定跑在 KingFlow 上，端点统一走 `https://www.kingflow.ai/v1`。

## 一、为什么需要一层桥接

先说清楚问题出在哪，很多人卡在这里以为是密钥或网络的问题，其实不是。

Codex 内部把对话组织成 Responses API 的请求结构，请求路径、消息容器、工具调用（tool call）的封装方式，以及流式（streaming）事件的分帧，都和经典的 Chat Completions 有差异。而 KingFlow 走的是 OpenAI 兼容的 Chat Completions 接口——这套接口生态最广、兼容性最好，绝大多数客户端都能直接吃。

问题因此变成：不是"能不能请求到 API"，而是"客户端发出的请求形态，和上游期望的形态不一致"。桥接层要做的，就是在本机把这一层差异抹平：

- **路径转换**：把 Codex 侧的请求路由映射到上游的 `/v1/chat/completions`。
- **字段转换**：请求体里的消息结构、参数命名做一次翻译。
- **流式转换**：把上游返回的 SSE（Server-Sent Events）数据帧，重新拼装成 Codex 能解析的流式事件。
- **工具调用转换**：function/tool call 的入参出参格式对齐，避免工具链在 Codex 里失效。

一句话：桥接层是一个跑在 `127.0.0.1` 上的翻译官，对 Codex 说 Responses API 的话，对 KingFlow 说 Chat Completions 的话。

## 二、配置目标：一张表理清两端

桥接方案里有两组地址和两把 Key，一定要分清楚，混了就报 401。左边是 Codex 看到的"假上游"（本机桥接），右边是桥接实际请求的"真上游"（KingFlow）。

| 位置 | 示例值 | 说明 |
|------|--------|------|
| Codex 侧 API 地址 | `http://127.0.0.1:4000/v1` | 指向本机桥接服务，Codex 只跟它对话 |
| Codex 侧 Key | `sk-proxy-local-replace-with-48-char-hex` | 本地代理认证，自己随便定，够长就行 |
| 桥接层上游 Base URL | `https://www.kingflow.ai/v1` | 桥接真正请求的模型服务端点 |
| 桥接层上游 Key | `sk-你的KingFlow密钥` | 从 KingFlow 后台密钥页复制 |

记住这个方向：**Codex → 本机 4000 → KingFlow**。Codex 侧的 Key 是桥接自己校验的本地口令，和 KingFlow 密钥没有任何关系；上游 Key 才是真正扣费、鉴权的那把。

## 三、.env 环境变量

桥接服务的配置全部收到 `.env` 里，别把密钥写死在代码或配置文件里。以下是一份最小可用配置：

```bash
# 本地代理认证口令，Codex 侧填的 Key 必须和它一致
PROXY_AUTH_KEY=sk-proxy-local-replace-with-48-char-hex

# 上游真实端点，统一走 KingFlow 兼容 /v1
UPSTREAM_BASE_URL=https://www.kingflow.ai/v1

# 上游密钥，从 KingFlow 后台复制，别泄露
UPSTREAM_API_KEY=sk-你的KingFlow密钥

# 默认模型，从模型列表里复制在售款
UPSTREAM_MODEL=gpt-5.5

# 桥接监听端口
PORT=4000
```

模型名一定用当前在售款，别照抄网上过期的名字。KingFlow 目前主流可选：`gpt-5.5`、`gpt-5.4`（GPT 系），以及经桥接可用的 `claude-opus-4-8`（旗舰）、`claude-sonnet-4-6`（均衡）、`claude-haiku-4-5`（高频低成本）。到底哪些能选，以模型列表接口返回为准。

## 四、最小验证流程：先跑通，再写进 Codex

不要一上来就改 Codex 配置，先在命令行把链路验通，出问题好定位。四步走：

**第一步：启动桥接服务，确认监听 4000。** 启动后控制台应打印类似 `listening on 127.0.0.1:4000` 的日志。如果端口被占，换一个（比如 4001）并同步改 `PORT`。

**第二步：请求模型列表。** 这一步验证"本机 → KingFlow"整条上游链路是否通，同时能拿到准确的在售模型名：

```bash
curl http://127.0.0.1:4000/v1/models \
  -H "Authorization: Bearer sk-proxy-local-replace-with-48-char-hex"
```

**第三步：发一次最短对话，看桥接日志。** 桥接日志里应该能依次看到：请求进入本地 → 转发到 `https://www.kingflow.ai/v1` → 上游返回 → 拼装后回给客户端。日志三段齐全，说明协议转换生效了。

```bash
curl http://127.0.0.1:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-proxy-local-replace-with-48-char-hex" \
  -d '{
    "model": "gpt-5.5",
    "messages": [{"role": "user", "content": "hello"}]
  }'
```

拿到正常 JSON 回复，就代表 Codex 侧口令、上游端点、上游密钥、模型名四项全对了。

**第四步：把 `http://127.0.0.1:4000/v1` 写进 Codex。** 命令行验通后再动 Codex 配置，见下一节。

排查时按这个顺序对：本地请求不到 → 服务没启动或端口被占；本地返回 401 → Codex 侧 Key 和 `PROXY_AUTH_KEY` 对不上；上游 401 → KingFlow 密钥不完整或复制漏了字符；`model_not_found` → 从模型列表重新复制模型名；能回复但流式卡死 → 桥接层的 SSE 转发没处理好。

## 五、把 4000 端口写进 Codex 官方 config

链路验通后，编辑 Codex 官方配置文件（`~/.codex/config.toml`），新增一个自定义 provider，把请求引到本机桥接：

```toml
model = "gpt-5.5"
model_provider = "kingflow"

[model_providers.kingflow]
name = "KingFlow via bridge"
base_url = "https://www.kingflow.ai/v1"
wire_api = "chat"
```

这里有两个新手最容易翻车的点，务必对齐：

1. **段名一致**：`model_provider = "kingflow"` 里的字符串，必须和 `[model_providers.kingflow]` 段名逐字一致，大小写、拼写都不能差。写成 `KingFlow` 或 `king_flow` 都会让 Codex 找不到 provider。
2. **base_url 指向 /v1**：`base_url = "https://www.kingflow.ai/v1"` 表示这条 provider 最终对应 KingFlow 兼容端点。若你选择让 Codex 直接打本机桥接，则把 `base_url` 换成 `http://127.0.0.1:4000/v1`；两种写法取决于你把协议转换放在桥接还是让 Codex 直连——桥接方案推荐前者，由本机代理统一收口。

上游密钥通过环境变量注入，Codex 会读取 `OPENAI_API_KEY`：

```bash
export OPENAI_API_KEY=sk-你的KingFlow密钥
```

保存后重启 Codex，用 `/model` 确认当前 provider 为 `kingflow`，随便问一句验证即可。

## 六、FAQ

**Q1：Codex 侧的 Key 和 KingFlow 密钥是同一个吗？**
不是。Codex 侧填的是桥接的本地口令（`PROXY_AUTH_KEY`），你自己定，只用于本机校验；KingFlow 密钥是上游真实密钥（`UPSTREAM_API_KEY`），负责真正的鉴权与计费。两者独立，别填反。

**Q2：为什么不能让 Codex 直连 KingFlow，非要加桥接？**
因为 Codex 走 Responses API，KingFlow 是 Chat Completions 兼容形态，请求路径、字段、流式分帧不一致。直连轻则报错，重则流式卡死或工具调用失效。桥接层把这层差异在本机抹平，是最稳的方案。

**Q3：模型名报 `model_not_found` 怎么办？**
先别改配置，用第四节的 `curl .../v1/models` 拉一次模型列表，从返回里复制准确的在售款（如 `gpt-5.5`、`claude-opus-4-8`），替换 `.env` 的 `UPSTREAM_MODEL` 和 Codex 的 `model`。名字差一个字符都会报错。

**Q4：能回复但流式（打字机效果）不动，卡在半截？**
这是桥接层 SSE 转发没处理好。确认桥接把上游返回的 `text/event-stream` 数据帧原样逐帧转出，而不是等全部收完再一次性吐。检查桥接日志里上游是否已开始返回流式帧——如果上游有帧、本地无帧，问题就在转发环节。

---

以上就是 Codex 接入 KingFlow 的完整桥接流程。核心记住三条：品牌端点统一 `https://www.kingflow.ai/v1`，Codex 段名 `kingflow` 前后一致，模型名只用在售款。链路先在命令行验通再写进 config，出问题按 401/model_not_found/流式三类分开排查，基本一次到位。
