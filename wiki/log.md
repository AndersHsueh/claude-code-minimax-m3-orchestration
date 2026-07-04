# Wiki Log

> 时间线日志。最后更新:2026-07-04
>
> 记录每次 **ingest**(新源摄取)/ **wiki**(页面更新)/ **meeting**(会议事件)/ **deliverable**(输出物生成)/ **engineering**(工程里程碑)/ **audit**(知识库审计)/ **lint**(健康检查)/ **todo**(待办)事件。
>
> 本文件由 LLM 在每次"新增 raw 源""重大 wiki 变更""里程碑达成"时**追加**条目,不删不改历史。

## 格式约定

每条以 `## [YYYY-MM-DD] type | short title` 开头,`type ∈ {ingest, wiki, meeting, deliverable, engineering, audit, lint, todo}`。

### grep 快捷命令

```bash
# 最近 5 条事件
grep "^## \[" wiki/log.md | tail -5

# 某天的全部事件
grep "2026-07-04" wiki/log.md

# 某一类型的所有事件
grep -E "ingest|wiki|engineering|meeting|audit|lint|todo" wiki/log.md

# 某个文件被引用的所有条目
grep "raw/<文件名>" wiki/log.md
```

### 维护纪律

1. **新源 → 必更**:每往 `raw/` 落一份新文件,**同一次会话内**必须追加一条 `ingest` 条目到本文件,并至少**更新 1 个 wiki 主题页**或在 `INDEX.md` 来源清单中登记
2. **wiki 变更 → 必登**:任何 wiki 页面新增/重写/纠正过时事实,追加一条 `wiki` 条目,说明动了哪个文件、为什么动
3. **工程里程碑 → 必登**:关键 PoC 完成、模块闭环、上线节点等,追加一条 `engineering` 条目
4. **输出物 → 必登**:`outputs/` 下任何正式交付物(docx/xlsx/html)生成时,追加一条 `deliverable` 条目
5. **lint → 必登**:每次跑 Lint 检查后,追加一条 `lint` 条目,说明扫描了什么、发现了什么、修复了什么
6. **todo → 必登**:发现 wiki 治理的待办项(如 INDEX 未同步、某主题待补),追加一条 `todo` 条目
7. **历史不删**:本文件为 append-only;如需纠正事实,加新条目而非修改旧条目

---

## [2026-07-04] ingest | raw/plan-1.md

双模型编排方案的原始设计文档。描述用 Opus 做规划、自建 MiniMax M3 做执行的八步落地方案(协议兼容确认、网关连通性测试、独立配置文件、可执行调用脚本、文本对话与工具调用测试、权限放行、CLAUDE.md 委派规则、实际使用流程),附排错清单。涉及主题:[[双模型编排方案概览]]、[[MiniMax-M3协议兼容与网关]]、[[claude-m3脚本与项目配置]]、[[执行委派规则]]。

## [2026-07-04] ingest | raw/2026-07-04-conversation.md

2026-07-04 落地会话的完整对话记录。涵盖协议兼容性测试(测试1)、工具调用测试(测试2)、`claude-m3` 脚本与项目权限配置落地、`CLAUDE.md` 委派规则撰写、端到端 bug 修复场景的委派与独立复查全过程。涉及主题:[[MiniMax-M3协议兼容与网关]]、[[claude-m3脚本与项目配置]]、[[执行委派规则]]、[[测试与端到端验证记录]]、[[待澄清与后续事项]]。

## [2026-07-04] engineering | claude-m3 委派链路端到端打通

完成 `plan-1.md` 八步方案的落地:MiniMax 官方 Anthropic 兼容端点验证通过、`claude-m3` 脚本与项目权限配置完成、`CLAUDE.md` 委派规则写入、端到端 bug 修复场景验证通过(主 session 独立复查确认,而非采信 M3 自我报告)。详见 [[测试与端到端验证记录]]。

## [2026-07-04] wiki | 首次搭建:从 raw 编译 6 个主题页

从 `raw/plan-1.md` 和 `raw/2026-07-04-conversation.md` 编译出 6 个 wiki 主题页:[[双模型编排方案概览]]、[[MiniMax-M3协议兼容与网关]]、[[claude-m3脚本与项目配置]]、[[执行委派规则]]、[[测试与端到端验证记录]]、[[待澄清与后续事项]]。

## [2026-07-04] wiki | INDEX.md 首次建立

建立 `wiki/INDEX.md`,含 6 个主题的主题清单(按"方案与架构 / 技术落地 / 验证与风险"分类)、2 个来源的来源清单、10 条关键事实速查、1 张主题关联图。

## [2026-07-04] lint | 首次自检:摘要/戳/链接密度/断链/orphan

扫描全部 9 个 wiki 文件。摘要覆盖 6/6(INDEX/log 按规则豁免),日期戳覆盖 9/9,链接密度全部 ≥5(最低 6,最高 17)。断链数 0。首次 orphan 扫描发现 `工作流-ingest-query-lint.md` 无 wiki 内 inbound 链接(仅被项目根 `CLAUDE.md` 引用),已在 `INDEX.md` 新增"工具与工作流"分类将其纳入主题清单(INDEX 兜底 inbound),复检后 orphan 数归零。

## [2026-07-04] deliverable | 发布为公开 GitHub 仓库

用户决定把本方案分享给更多人。新增根目录 `README.md`(英文,面向国际读者,概述方案动机、快速上手步骤、已验证结果、已知局限,并指向 `wiki/INDEX.md` 获取完整中文叙事)和 `.gitignore`。推送前扫描全仓库确认无真实密钥/邮箱泄漏(`raw/plan-1.md` 里的 token 是占位符 `dummy-or-your-gateway-token`,非真实值)。仓库地址:<https://github.com/AndersHsueh/claude-code-minimax-m3-orchestration>(public)。
