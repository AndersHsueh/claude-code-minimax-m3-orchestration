# 对话记录:MiniMax M3 双模型编排方案落地

**日期**:2026-07-04
**项目目录**:`/Users/xueyuheng/research/claude-with-mmx`
**参与方**:用户(老友) + Claude Code(Sonnet 5)

---

## 背景

用户此前已写好 `plan-1.md`——一套双模型编排方案:用 Claude Pro 订阅的 Opus 做
"制定计划",把重复执行、多轮工具调用的活转移给自建的 MiniMax M3,从而节省
Anthropic 的 5 小时额度。方案分八步:确认协议兼容 → 测通网关 → 建独立 settings →
建可执行脚本 `claude-m3` → 测试文本对话与工具调用 → 放行权限 → 写 CLAUDE.md
委派规则 → 实际使用。

## 对话过程

### 1. 确认现状

用户开场介绍今天要做的事,并指出 MiniMax M3 对 Anthropic 协议兼容极好,这是选择
它的原因之一。让 Claude 用 `claude --settings ~/.claude/settings-mmx.json -p XXXX`
检测协议这一关。

Claude 读取了 `~/.claude/settings-mmx.json`,发现它直接指向 MiniMax 官方的
Anthropic 兼容端点 `https://api.minimaxi.com/anthropic`(而非自建网关),说明
plan-1.md 里第一、二步的"翻译层"问题已经被 MiniMax 官方解决。

### 2. 测试 1:文本对话

```bash
claude --settings ~/.claude/settings-mmx.json -p "用一句话介绍你自己" --output-format json
```

结果:成功。`result` 字段返回 `"我是 Claude Code,由 MiniMax-M3 模型驱动,是你的
AI 软件工程助手。"`,`modelUsage` 正确标注为 `MiniMax-M3[1m]`。首次调用 ttft 高达
176 秒(推测是冷启动/缓存问题)。

### 3. 用户提醒:工具调用必须带 --allowedTools

用户提到"昨天 fable 说到一个问题",工具调用在非交互模式下如果需要人工确认会导致
进程卡死/中断,必须显式传 `--allowedTools`。Claude 先用 `claude --help` 确认了
正确的参数名(`--allowedTools` / `--allowed-tools`),再进行测试。

### 4. 测试 2:工具调用

```bash
cd /tmp/mmx-test && claude --settings ~/.claude/settings-mmx.json \
  -p "在当前目录创建一个 test.txt 文件,内容写 hello from M3" \
  --allowedTools "Write" --output-format json
```

结果:成功,12 秒完成,`permission_denials` 为空,文件真实落地且内容正确。

### 5. 落地第四、六、七步

- **第四步**:创建 `~/bin/claude-m3` 可执行脚本(内容:`exec claude --settings
  ~/.claude/settings-mmx.json "$@"`),`chmod +x`,验证已在 PATH 中,冒烟测试
  6.8 秒返回 `"OK"`。
- **第六步**:在项目 `.claude/settings.json` 里加 `Bash(claude-m3:*)` 权限放行
  (备注:用户全局 settings.json 里 `Bash` 已无限制放行,这条更多是显式声明,便于
  项目独立分享或全局配置收紧时仍生效)。
- **第七步**:在项目根目录写 `CLAUDE.md`,定义委派规则——执行阶段任务交给
  `claude-m3`,规划/架构阶段自己做。额外补充了三条实测经验:全新会话无记忆、
  必须带 `--allowedTools`、首次调用 ttft 可能高达 100+ 秒。

### 6. 端到端验证

设计了一个真实场景:在 `/tmp/mmx-e2e-test/` 埋一个 bug(`calc.py` 里 `subtract`
函数误写成 `a + b`),让 `claude-m3` 独立读文件、定位 bug、修复、跑测试自我验证。

委派 prompt:

```
你在目录 /tmp/mmx-e2e-test/ 下工作,里面有 calc.py 和 test_calc.py 两个文件。
运行 python3 test_calc.py 目前会因为 AssertionError 失败。请你:
1) 读取 calc.py 和 test_calc.py 找出具体是哪个函数有 bug;
2) 修复 calc.py 里的 bug(不要改动 test_calc.py);
3) 重新运行 python3 test_calc.py 确认能看到 ALL TESTS PASSED 且退出码为 0 之后再结束任务。
```

`claude-m3` 用 5 轮工具调用、25 秒、$0.34 完成任务,自我报告修复成功。Claude 没有
直接采信这份报告,而是独立复查:

- `calc.py` 确认改成了 `return a - b`
- `test_calc.py` 内容与委派前完全一致(未被误改)
- 独立重跑 `python3 test_calc.py`:输出 `ALL TESTS PASSED`,退出码 `0`

**结论**:端到端闭环验证通过——主 session 埋任务上下文 → `claude-m3` 独立执行 →
主 session 独立复查确认,全程未消耗 Anthropic 额度。

## 产出文件清单

- `~/bin/claude-m3`(可执行脚本)
- `/Users/xueyuheng/research/claude-with-mmx/.claude/settings.json`(项目权限)
- `/Users/xueyuheng/research/claude-with-mmx/CLAUDE.md`(委派规则)
- `/tmp/mmx-e2e-test/`(端到端验证用的临时测试文件,未纳入项目版本管理)

## 尚待验证/后续可做

- plan-1.md 第一、二步(网关自建 / vLLM serve)在本次对话里未涉及,因为用户直接
  使用了 MiniMax 官方 Anthropic 兼容端点,等同于绕过了这两步。
- 首次调用 176 秒的高延迟原因未深挖(冷启动?服务端排队?),如果后续批量委派时
  频繁出现,需要进一步排查。
