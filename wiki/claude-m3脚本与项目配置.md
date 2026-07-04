# claude-m3脚本与项目配置

> 最后更新:2026-07-04

## 摘要

为了让主 session 能一条命令委派任务给 MiniMax M3,落地了可执行脚本 `~/bin/claude-m3`(而非 shell alias),并在项目 `.claude/settings.json` 里放行了 `Bash(claude-m3:*)` 权限,避免每次调用都跳出确认框。脚本与权限均已通过冒烟测试。

## 一、为什么不能用 alias

`raw/plan-1.md` 明确指出:不要用 `~/.zshrc` 里的 `alias` 来做这件事,因为 Claude Code 主 session 执行 Bash 工具时,通常起的是非交互、非登录的子 shell,不会加载 `.zshrc`,alias 不会生效。必须用一个**真正的可执行文件**。

## 二、~/bin/claude-m3 脚本内容

```bash
#!/bin/bash
exec claude --settings ~/.claude/settings-mmx.json "$@"
```

- 指向已验证可用的 `~/.claude/settings-mmx.json`(而非 plan-1.md 里的占位文件名 `settings-m3.json`),因为该配置已经指向 MiniMax 官方 Anthropic 兼容端点,详见 [[MiniMax-M3协议兼容与网关]]。
- 已 `chmod +x`,且 `~/bin` 本就在用户 PATH 中,无需额外配置。

## 三、项目权限放行(.claude/settings.json)

项目根目录 `/Users/xueyuheng/research/claude-with-mmx/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": ["Bash(claude-m3:*)"]
  }
}
```

> 备注:用户全局 `~/.claude/settings.json` 里 `Bash` 权限本就是无限制放行(`"allow": ["Bash", ...]`),所以这条项目级权限严格来说是冗余的。保留它是为了显式声明——万一未来全局权限收紧,或项目被单独分享给他人使用,这条规则依然生效。

## 四、冒烟测试结果

```bash
claude-m3 -p "回复 OK 即可" --output-format json
```

6.8 秒返回 `"OK"`,`permission_denials` 为空,证明脚本本身、权限配置、协议链路三者都工作正常。

## 关联主题

[[双模型编排方案概览]] [[MiniMax-M3协议兼容与网关]] [[执行委派规则]] [[测试与端到端验证记录]] [[待澄清与后续事项]]

## 来源

- `raw/plan-1.md` 第四、六步
- `raw/2026-07-04-conversation.md` 第 5 节
