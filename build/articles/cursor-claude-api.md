<div align="center">
<h3>KingFlow · 国内直连 AI API 中转</h3>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/官网-www.kingflow.ai-FF6B35" alt="KingFlow"></a>
<a href="https://www.kingflow.ai/v1"><img src="https://img.shields.io/badge/兼容端点-/v1-2EA44F" alt="OpenAI兼容"></a>
<a href="https://www.kingflow.ai"><img src="https://img.shields.io/badge/模型-Claude%20Opus%204.8%20%7C%20GPT--5.5-8957E5" alt="模型"></a>
</div>

# Cursor 接入 Claude/GPT 中转 API 实战教程（国内可用）

用 Cursor 写代码这一年多，最折磨我的从来不是它的功能，而是「怎么让它在国内稳定地跑起来」。官方订阅要绑境外卡，自己开 API Key 又要折腾网络，token 烧得还快。后来我把整套工作流切到了 KingFlow 中转上，配置就改一个 Base URL，该用 Opus 4.8 用 Opus 4.8，该用 GPT-5.5 用 GPT-5.5，一点不打折扣。这篇就把 Cursor 接入中转 API 的完整步骤、代码示例、对账方法和选型建议一次写清楚，照着抄就能用。

## 一、Cursor 用官方 API 的几个痛点

如果你也踩过下面这些坑，这篇文章就是写给你的：

1. **境外卡门槛高**。Cursor Pro 订阅和 OpenAI/Anthropic 官方 API 都要求绑定境外信用卡，很多人卡在第一步，要么没卡，要么卡被风控。
2. **汇率与隐性损耗**。美元计费，每次扣款都要吃一道汇率差，月底账单换算成人民币往往比预期高一截。
3. **访问不稳定**。直连官方端点在国内丢包、超时是常态，Cursor 的补全和 Chat 经常转圈，Agent 模式跑到一半断流，体验很碎。
4. **token 烧得快、成本不可控**。Cursor 的 Agent 和 Composer 一次任务能塞进去几万 token 的上下文，官方按美元原价计费，长项目下来钱包压力很大，而且你很难提前知道这一单到底花了多少。

说白了，痛点集中在「支付门槛 + 网络稳定性 + 成本透明度」三件事上。

## 二、为什么用 KingFlow 中转

我选 KingFlow 的核心理由就三条，正好对应上面的痛点：

- **倍率透明，成本看得见**。官网直接公示各模型倍率，Claude Opus 4.8、GPT-5.5 这类强模型的输入价大约 ¥0.5/1M tokens 起，输出、图片、工具调用费用都列得清清楚楚。后台还能查到每一次调用的明细，对账不靠猜。对 Cursor 这种动辄长上下文的场景特别友好。
- **国内直连，流式稳定**。不用额外配网络环境，Cursor 的代码补全、Chat、Agent 流式输出都很顺，跑长任务不容易断。
- **模型保真，不掉包**。OpenAI 全系（GPT-5.4 / GPT-5.5）和 Anthropic Claude（claude-opus-4-8 / claude-sonnet-4-6 / claude-haiku-4-5）都覆盖，实测返回质量接近官方，没遇到过拿小模型冒充强模型的情况——这点在中转圈里其实挺难得。
- **支付省心**。人民币付费，免去绑境外卡和汇率折腾。

## 三、Cursor 配置步骤

Cursor 兼容 OpenAI 接口，所以接入中转本质上就是「改端点 + 填 Key + 选模型」三步。

1. 打开 Cursor，进入 **Settings → Models**（或 Cursor Settings 里的 Models 面板）。
2. 找到 **OpenAI API Key** 区域，把开关打开，填入你在 KingFlow 后台生成的 API Key。
3. 展开 **Override OpenAI Base URL**，填入：

   ```
   https://www.kingflow.ai/v1
   ```

4. 在模型列表里勾选你要用的模型。如果默认列表里没有，点 **Add model** 手动添加模型名，例如：

   - `claude-opus-4-8`（旗舰，复杂重构、架构推理首选）
   - `gpt-5.5`（综合能力强，日常编码和 Agent 都能扛）
   - `claude-sonnet-4-6`（均衡，速度和质量折中）

5. 点 **Verify** 验证连通。验证通过后，回到编辑器右下角的模型选择器，切到刚加的模型即可。

> 小提示：Override Base URL 之后，Cursor 的请求会全部走 KingFlow 端点。如果你既要用官方又要用中转，建议用两套 Cursor 配置档或在切换时手动改 Base URL，避免混淆计费来源。

配置完，随手让它补全一段代码或者在 Chat 里问一句，能正常流式返回就说明通了。

## 四、代码侧 OpenAI SDK 示例

Cursor 之外，你本地的脚本、CI、Agent 也可以共用同一个 Key 和端点。只要是 OpenAI 兼容 SDK，改一行 `base_url` 就行，其余代码完全不动：

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-你的KingFlowKey",
    base_url="https://www.kingflow.ai/v1",
)

resp = client.chat.completions.create(
    model="gpt-5.5",
    messages=[{"role": "user", "content": "用 Python 写一个快速排序"}],
)
print(resp.choices[0].message.content)
```

想调 Claude 的话，连代码都不用改结构，只把 `model` 换成 `claude-opus-4-8` 即可：

```python
resp = client.chat.completions.create(
    model="claude-opus-4-8",
    messages=[{"role": "user", "content": "帮我审查这段代码的并发安全问题"}],
)
```

用 cURL 快速自测连通性也很方便：

```bash
curl https://www.kingflow.ai/v1/chat/completions \
  -H "Authorization: Bearer sk-你的KingFlowKey" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.5",
    "messages": [{"role": "user", "content": "ping"}]
  }'
```

能拿到正常 JSON 返回，就说明 Key 和端点都没问题，再回头配 Cursor 一次过。

## 五、倍率透明与对账

中转用得安不安心，最后都落到「钱花在哪」这件事上。KingFlow 后台的用量明细这块做得比较踏实：

- 每次调用都能看到对应的模型、输入/输出 token 数和扣费金额，按时间倒序排列。
- 倍率公示在官网，输入、输出、图片、工具调用分开计价，自己心里能算明白一单大概多少钱。
- 对 Cursor 这种「一次 Agent 任务吃几万 token」的用法，建议每天扫一眼明细，能很快定位是哪个项目、哪类操作在烧 token，进而优化上下文长度。

我自己的习惯是：跑大重构前先在便宜模型上验证一遍提示词，确认思路对了再切 Opus 4.8 上真任务，账单立刻就下来了。

## 六、选型建议

不同场景模型选法不一样，给个我自己在用的对照：

| 场景 | 推荐模型 | 说明 |
|------|----------|------|
| 生产核心业务 | 官方 API / 云厂商 MaaS 为主，中转做灾备 | 关键链路别单点依赖任何一家 |
| 个人开发 / 日常编码 | gpt-5.5 或 claude-sonnet-4-6 | 性价比高，补全和 Chat 都够用 |
| 复杂重构 / 架构推理 | claude-opus-4-8 | 长上下文、深推理时质量最稳 |
| 高频小任务 / 批量 | claude-haiku-4-5 | 便宜快，适合 lint、注释、改名 |
| 长期项目 | 主用中转 + 备 1-2 个渠道 | 避免单点故障，关键时刻能切换 |

核心原则一句话：**倍率透明 + 模型保真 + 访问稳定**，三者都满足才值得长期用。

## 七、FAQ

**Q1：接入 KingFlow 后还需要额外的网络环境吗？**
不需要。KingFlow 是国内直连，Cursor 的补全、Chat、Agent 流式输出都能直接跑，省去了配网络的麻烦。

**Q2：Override Base URL 后，Cursor 原来的内置模型还能用吗？**
改了 Base URL 后请求会统一走 KingFlow 端点。如果你还想用官方内置功能，建议另存一套配置或在切换时手动改回，避免计费来源混在一起。

**Q3：会不会出现模型被降级、拿小模型冒充的情况？**
实测 Claude 和 GPT 全系返回质量接近官方，没遇到过掉包。担心的话，第一次接入可以用同一个提示词分别问官方和中转，对比一下输出就放心了。

**Q4：Cursor 里填的 API Key 和代码里的是同一个吗？**
是同一套。KingFlow 后台生成的 Key 在 Cursor、本地 SDK、cURL、其他 OpenAI 兼容客户端里通用，用量也会汇总到同一个后台明细里，方便统一对账。

---

总结一下：Cursor 接中转其实没什么玄学，把 Base URL 改成 `https://www.kingflow.ai/v1`、填好 Key、选好模型这三步走完就能用。剩下的功夫花在「看明细、控成本、选对模型」上，长期下来既稳又省心。
