# Claude Code 双模型编排方案:Opus 规划 + 自建 MiniMax M3 执行

## 目标

用 Claude Pro 订阅的 Opus 做"制定计划 / 写 workflow 脚本"这类高价值、低频次的事,
把"反复执行 / 多轮工具调用 / 长时间迭代"这类烧额度的事,转移给自建 GPU 服务器上的
MiniMax M3,从而不消耗(或极少消耗)Anthropic 的 5 小时额度。

整体结构:

```
主 session(Claude Pro, 手动切到 Opus)
  └─ 只负责:理解需求 → 拆解任务 → 写出一份可执行的 workflow 脚本/计划
       ↓
claude-m3(独立的 CLI 别名,指向你自己的 M3 服务器,不占 Anthropic 额度)
  └─ 负责:按脚本反复执行 → 读文件、改代码、跑测试、修 bug、提交
```

---

## 第一步:确认 M3 能被"当 Claude Code 的后端"调用

**关键前提**:Claude Code 只会用 Anthropic 的 Messages API 协议(`/v1/messages`)发请求。
MiniMax M3 本身的 API 格式不是这个协议,所以中间必须有一层"翻译层",把 Claude Code
发来的 Anthropic 格式请求,转换成 M3 能听懂的格式,再把 M3 的回复转换回来。

常见的两种做法(任选其一):

1. **用支持 Anthropic Messages API 的推理框架直接serve M3**
   (例如 vLLM 这类支持该协议的框架,前提是你的部署环境里 vLLM 版本支持这个 API 形态,
   且开启了工具调用相关参数,如 `--enable-auto-tool-choice` 和对应的 `--tool-call-parser`)。

2. **自己写一个小型转换网关**
   接收 `/v1/messages` 格式的请求 → 转换成 MiniMax 官方 API 格式发给 M3 → 
   再把 M3 的回复转换回 Anthropic 格式返回。

> ⚠️ **验证要点**:无论用哪种方式,部署完成后必须先做一次"工具调用"的真实测试
> (见第五步),确认返回的 `tool_use` 结构是 Claude Code 能正确解析的,而不是
> 只测过普通文本对话就直接上生产流程。

---

## 第二步:确认网关服务正常运行

假设你的转换网关(或 vLLM)监听在 `http://<你的服务器IP>:8000`,先脱离 Claude Code
单独测试这个地址能不能返回符合 Anthropic 格式的响应,比如用 curl 发一个最简单的请求
确认能收到 200 响应和正确的 JSON 结构。**这一步一定要先过,再进行下一步**,否则后面
所有配置都是在一个不通的地址上瞎折腾。

---

## 第三步:创建独立的 Claude Code 配置文件

新建一个专门指向 M3 的 settings 文件(路径和文件名可以自定,这里用示例名):

```bash
mkdir -p ~/.claude
```

创建文件 `~/.claude/settings-m3.json`,内容:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://<你的服务器IP>:8000",
    "ANTHROPIC_AUTH_TOKEN": "dummy-or-your-gateway-token",
    "ANTHROPIC_MODEL": "minimax-m3"
  }
}
```

字段说明:
- `ANTHROPIC_BASE_URL`:你的网关/vLLM 服务地址,替换成实际 IP 和端口。
- `ANTHROPIC_AUTH_TOKEN`:如果你的网关不校验鉴权,填任意占位字符串;
  如果网关有自己的鉴权机制,填真实 token。
- `ANTHROPIC_MODEL`:填你的网关/vLLM 对外暴露的模型名称(需要和网关里配置的名字一致)。

---

## 第四步:创建可执行的调用脚本(不要用 shell alias)

**重要**:不要用 `~/.zshrc` 里的 `alias` 来做这件事。因为 Claude Code 主 session
执行 Bash 工具时,通常起的是非交互、非登录的子 shell,不会加载 `.zshrc`,alias
不会生效。必须用一个**真正的可执行文件**。

```bash
mkdir -p ~/bin
```

创建文件 `~/bin/claude-m3`,内容:

```bash
#!/bin/bash
exec claude --settings ~/.claude/settings-m3.json "$@"
```

赋予执行权限:

```bash
chmod +x ~/bin/claude-m3
```

确认 `~/bin` 在 PATH 里(如果不在,加到 `~/.zshrc` 或 `~/.bashrc` 里):

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

验证脚本能被找到:

```bash
which claude-m3
```

应该输出 `~/bin/claude-m3` 的完整路径。如果没有输出,说明 PATH 没配对,回头检查。

---

## 第五步:单独测试 claude-m3 能否正常工作(先测简单对话,再测工具调用)

**测试 1:最简单的文本对话**

```bash
claude-m3 -p "用一句话介绍你自己" --output-format json
```

预期:能拿到一个 JSON,里面 `result` 字段是 M3 的回复,`session_id` 等元数据齐全。
如果这一步就报错(连接失败、超时、格式错误),先回去检查第二步的网关是否真的通。

**测试 2:工具调用是否可靠**(这是最关键的一步,直接决定这套方案能不能用)

```bash
claude-m3 -p "在当前目录创建一个 test.txt 文件,内容写 hello" --allowedTools "Write"
```

预期:M3 能正确发起一次 `Write` 工具调用,文件被成功创建。

**如果测试 2 失败或不稳定**:说明网关的工具调用转换有问题,或者 M3 服务端没有正确
开启工具调用支持,需要回到第一步重新检查网关/vLLM 的工具调用配置,不要跳过这一步
直接进入生产使用。

---

## 第六步:给主 session(Opus)配置权限,允许它调用 claude-m3

在你的项目目录下(或 `~/.claude/settings.json` 全局配置里),添加权限放行,
否则每次调用都会跳出确认框,打断自动化流程:

```json
{
  "permissions": {
    "allow": ["Bash(claude-m3:*)"]
  }
}
```

---

## 第七步:在项目根目录写 `CLAUDE.md`,定义委派规则

创建(或编辑)项目根目录下的 `CLAUDE.md`,加入类似下面的规则段落:

```markdown
## 执行委派规则

当任务进入"具体执行阶段"(比如:按已有计划批量改文件、跑测试、
修复报错、反复迭代验证)时,不要自己执行,而是调用本地部署的
MiniMax M3 来完成,命令格式如下:

    claude-m3 -p "<具体任务描述,把上下文写清楚,因为它是全新会话>" \
      --output-format json \
      --allowedTools "Read,Write,Edit,Bash"

拿到返回的 JSON 后,解析 `.result` 字段作为这一步的执行结果,
判断是否需要进入下一轮迭代。

当任务属于"制定方案 / 架构设计 / 复杂推理判断"阶段时,由你(主模型)
自己直接完成,不要委派。
```

> 💡 **提示**:M3 每次被调用都是全新会话,不会记得之前发生了什么。
> 委派的 prompt 里必须把当前上下文(要改哪个文件、目标是什么、
> 上一步做了什么)写清楚,不能假设它"知道"之前的进度。

---

## 第八步:实际使用流程

1. 打开主 session,手动切换到 Opus:`/model opus`
2. 描述你的大任务,让 Opus 做需求拆解、架构设计,产出一份具体的执行计划
   (可以是文字步骤,也可以是一个脚本文件)
3. 计划完成后,**立刻把模型切回 Sonnet**(`/model sonnet`),避免后续琐碎交互
   继续消耗 Opus 额度
4. 让主 session(此时已是 Sonnet,或者你也可以直接手动跑)按照 `CLAUDE.md`
   里的规则,把执行阶段的任务通过 `claude-m3` 委派出去
5. M3 反复执行、迭代、验证,过程中不占用你的 Anthropic 额度
6. 执行完成后回到主 session,做最终 review 和确认

---

## 排错清单(如果哪一步不通,按这个顺序查)

1. 网关/vLLM 服务本身是否正常响应(第二步)
2. `settings-m3.json` 里的 `ANTHROPIC_BASE_URL` 是否正确、端口是否开放
3. `claude-m3` 脚本是否有执行权限、PATH 是否配置正确
4. 简单文本对话(测试1)能不能过 —— 过不了说明连通性有问题
5. 工具调用(测试2)能不能过 —— 过不了说明网关的协议转换/工具调用配置有问题
6. 主 session 的权限设置是否放行了 `Bash(claude-m3:*)`,否则会一直卡在确认框
7. `CLAUDE.md` 的委派规则是否被正确读取(可以直接问主 session "你现在的委派规则是什么")

