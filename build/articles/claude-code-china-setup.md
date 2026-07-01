<div align="center">
<h3>KingFlow · 国内直连 AI API 中转</h3>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
</div>

# Claude Code 国内配置实录：settings.json 一步接入 + Prompt Cache 透传

折腾 Claude Code 大半年，被问最多的一句话是"国内到底怎么配才能跑起来、还不掉缓存"。这篇不讲虚的，就当我把自己 `~/.claude/settings.json` 摊开给你看，一个文件从头配到尾，再教你验证 Prompt Cache 是不是真的透传下来了——这一步很多人漏掉，结果白白多花几倍的钱。

## 一、直连 Anthropic 在国内为什么走不通：三关

先说清楚为什么不能直接连官方，省得你去踩我踩过的坑。

**注册关。** Anthropic 账号要绑美区信用卡，国内双币 Visa/Master 的 BIN 段几乎全被拒。我拿三张不同银行的卡试过，全部支付失败，最后只能作罢。

**网络关。** Anthropic 对国内 IP 有限制。Claude Code 不是问一句答一句，它是高频长连接的 Agent，读文件、跑命令、看报错反复往返，机场代理的出口 IP 三五分钟就暴露，连发 403/429。你正重构到一半突然断流，那种感觉很崩。

**风控关。** 设备指纹加出口 IP 频繁切换，很容易被判定异常直接冻结账号，然后系统自动退费——号没了，钱退回来，活儿还没干。

三关叠加，直连这条路对绝大多数国内开发者是堵死的。

## 二、为什么我选国内中转（KingFlow）

摆在面前其实只有两条路。

一条是官方 API 加自建专线代理：你得先有美区身份和支付能力,再租一条稳定的住宅或商业专线代理,而且代理必须**透明转发原始 request body**——任何对 `system`、`messages`、`cache_control` 的解析重组,都会破坏 Prompt Cache 的哈希一致性,缓存直接失效。专线月租一两百,加上 API 费用,算下来不便宜,维护也累。

另一条，也是我现在长期在用的，就是走国内 Claude API 中转，用的是 [KingFlow](https://www.kingflow.ai)。选它的核心理由是：它走的是 Anthropic 官方 `/v1/messages` 协议，不是那种反代 Cursor/Kiro 逆向出来的接口。逆向接口有两个致命问题——不支持 Prompt Cache，而且 Anthropic 一更新协议就挂。官方协议这条底线，我认为是评估任何中转站的第一道门槛。

## 三、核心：只改 `~/.claude/settings.json` 一个文件

接入真的只动一个文件，不用改环境变量、不用装插件。完整配置贴在下面，直接抄：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://www.kingflow.ai",
    "ANTHROPIC_AUTH_TOKEN": "在 KingFlow 后台领取的 API Key",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0"
  },
  "effortLevel": "medium"
}
```

逐个参数说清楚，别照抄了还不知道每一项在干嘛：

- **`ANTHROPIC_BASE_URL`**：把请求端点从官方换成 `https://www.kingflow.ai`。这一行是整个接入的关键，Claude Code 所有请求都会打到这里。注意是根域名，不带 `/v1`——Claude Code 自己会拼路径。
- **`ANTHROPIC_AUTH_TOKEN`**：鉴权用的 Key，登录 KingFlow 控制台生成后填进来。它替代了官方的 `ANTHROPIC_API_KEY`，两者不要同时设，否则 Claude Code 可能优先读错那个。
- **`API_TIMEOUT_MS`**：单次请求超时，这里给到 `3000000`（50 分钟）。别嫌大，Claude Code 跑跨文件大重构时一个回合真能跑很久，默认超时会把长任务掐断，改大了有备无患。
- **`CLAUDE_CODE_ATTRIBUTION_HEADER`**：设成 `"0"` 关掉 commit 里的自动署名。除了干净，还有个实际好处——减少注入到请求里的动态内容，**提高 Prompt Cache 命中率**。
- **`effortLevel`**：设 `medium`，日常开发感知不到差别，却能省 20%-30% 的推理 token。要啃硬架构题临时调 `high` 就行。

存盘，重开 Claude Code，直接就能用了。

## 四、Prompt Cache 到底有没有真透传？教你验证

这是全文最该记住的一段。Claude Code 的输入 token 远大于输出（它每次都要把大段上下文喂进去），Prompt Cache 能把成本砍掉 50%-90%，缓存部分按 Anthropic 官方 0.1x 输入价计费。所以中转站有没有把缓存**原样透传**，直接决定你的账单。

验证方法很朴素：构造一个带 `cache_control` 的请求，**连发两次**，看第二次返回的 `usage.cache_read_input_tokens` 是不是非零。

```bash
curl https://www.kingflow.ai/v1/messages \
  -H "x-api-key: $ANTHROPIC_AUTH_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 64,
    "system": [
      {"type": "text", "text": "这里放一大段稳定的系统提示，长度要够，让它值得被缓存……",
       "cache_control": {"type": "ephemeral"}}
    ],
    "messages": [{"role": "user", "content": "ping"}]
  }'
```

第一次返回你会看到 `cache_creation_input_tokens` 有值（写缓存），`cache_read_input_tokens` 为 0。**紧接着原样再发一次**，第二次若 `cache_read_input_tokens` 变成非零，说明缓存被官方接住并透传回来了——这才是真透传。如果两次都是 0，那这家中转多半在中间重组了 request body，缓存哈希对不上，趁早换。我在 KingFlow 上这一步是过的，第二次稳定读到缓存。

## 五、延迟与汇率对比：为什么国内节点省心又省钱

同样一次首字响应（TTFT），服务器位置差出来的体感是数量级的：

| 节点位置 | TTFT（首字延迟） | 是否需要代理 | 综合体验 |
|---|---|---|---|
| 美国 California | 40-50s | 需稳定专线 | 长任务频繁掉线 |
| 日本 Tokyo | 15-25s | 需代理 | 可用但偶尔卡顿 |
| 国内节点（KingFlow） | 1-3s | 无需代理 | 顺滑，接近本地 |

延迟之外还有一笔隐性账——**内部汇率**。不同平台的计价差价能到 3 倍，也就是同样充 100 块，能跑的请求数可能差三倍。这块建议你别只看标价，用第四节的方法实测几天缓存命中，再结合汇率算真实单价。

## 六、FAQ

**Q1：`ANTHROPIC_BASE_URL` 要不要带 `/v1`？**
Claude Code 里填根域名 `https://www.kingflow.ai` 就行，客户端会自己拼 `/v1/messages`。但你手动跑 cURL 验证缓存时，路径要写全 `https://www.kingflow.ai/v1/messages`。

**Q2：配好了报 401 怎么办？**
先确认没有残留的 `ANTHROPIC_API_KEY` 环境变量在跟 `settings.json` 里的 `ANTHROPIC_AUTH_TOKEN` 打架。清掉旧环境变量、重开终端，九成能解决。

**Q3：能路由到别的模型吗？**
可以。Claude Code 走 Anthropic 协议，日常小改用 `claude-haiku-4-5` 压成本，中等任务 `claude-sonnet-4-6`，跨文件大重构或架构决策再上 `claude-opus-4-8`，`/model` 随时切，混着用比全程旗舰省不少。

**Q4：`API_TIMEOUT_MS` 设 50 分钟会不会太夸张？**
不会。这只是**上限**，短任务该秒回还是秒回，它只在长任务真跑到那么久时兜底，避免被中途掐断。留大一点纯赚安心。

## 收个尾

整套流程就一句话：改 `~/.claude/settings.json` 一个文件接入 KingFlow，然后用连发两次看 `cache_read_input_tokens` 的办法确认 Prompt Cache 真透传。国内节点 1-3s 的 TTFT、不用挂代理、缓存不掉、汇率友好，对我这种每天泡在终端里的人来说，这就是最省心的组合。建议先小额充值，把上面的验证跑通、把一周的真实工作流跑顺，再决定要不要长期用。
