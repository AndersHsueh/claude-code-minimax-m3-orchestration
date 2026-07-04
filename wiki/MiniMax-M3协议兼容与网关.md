# MiniMax-M3协议兼容与网关

> 最后更新:2026-07-04

## 摘要

`raw/plan-1.md` 原方案设计认为需要自建"翻译层"(vLLM 或自写网关)把 Claude Code 发出的 Anthropic Messages API 请求转换成 MiniMax 格式,并给出了独立验证步骤(方案第一、二步)。但实际落地时发现 MiniMax 官方已直接提供 Anthropic 协议兼容端点,`settings-mmx.json` 直接指向该端点,完全绕过了自建网关这两步。文本对话测试(测试 1)已通过,但首次调用延迟高达 176 秒,原因未查明。

## 一、原方案设计(plan-1.md 第一、二步)

方案设想两种翻译层做法:

1. 用支持 Anthropic Messages API 的推理框架直接 serve M3(如 vLLM,需开启 `--enable-auto-tool-choice` 等工具调用参数)
2. 自己写一个小型转换网关,做 `/v1/messages` ↔ MiniMax 官方 API 格式互转

并要求先用 curl 单独测通网关(200 响应 + 正确 JSON 结构),再进行后续配置。

## 二、实际情况:官方兼容端点

用户已有配置文件 `~/.claude/settings-mmx.json`,其中:

```json
"ANTHROPIC_BASE_URL": "https://api.minimaxi.com/anthropic"
```

这是 MiniMax **官方**提供的 Anthropic 协议兼容端点,不是自建网关。也就是说方案第一、二步(翻译层搭建 + 网关连通性测试)在本项目里**已经被官方解决,无需用户自建**。

> 已过时提示:`raw/plan-1.md` 第一、二步描述的自建网关流程,在使用 MiniMax 官方端点时不适用,仅在未来切换到其它不提供 Anthropic 兼容层的模型时才需要参考。

## 三、settings-mmx.json 关键字段

| 字段 | 值 | 说明 |
|------|-----|------|
| `ANTHROPIC_BASE_URL` | `https://api.minimaxi.com/anthropic` | 官方 Anthropic 兼容端点 |
| `ANTHROPIC_AUTH_TOKEN` | `sk-cp-pp_...`(真实 token) | MiniMax 平台密钥 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `MiniMax-M3` | Opus 请求映射到 M3 |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` / `HAIKU_MODEL` | `MiniMax-M2.7-highspeed` | 低算力请求映射到轻量模型 |
| `API_TIMEOUT_MS` | `3000000` | 超时放宽到 3000 秒,应对高延迟 |

## 四、测试1结果:文本对话与延迟问题

命令:

```bash
claude --settings ~/.claude/settings-mmx.json -p "用一句话介绍你自己" --output-format json
```

结果:成功,`result` 字段正常返回,`modelUsage` 正确标注为 `MiniMax-M3[1m]`(百万 token 上下文窗口)。**但首次调用 ttft 长达 176 秒**,推测是冷启动或服务端排队,原因未深挖,详见 [[待澄清与后续事项]]。后续调用(冒烟测试、端到端测试)ttft 都在 6-8 秒区间,说明 176 秒可能是偶发的冷启动现象而非常态,但尚未有足够样本确认。

## 关联主题

[[双模型编排方案概览]] [[claude-m3脚本与项目配置]] [[测试与端到端验证记录]] [[执行委派规则]] [[待澄清与后续事项]]

## 来源

- `raw/plan-1.md` 第一、二步
- `raw/2026-07-04-conversation.md` 第 1-2 节
