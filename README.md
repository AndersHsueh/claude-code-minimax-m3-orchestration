# Claude Code × MiniMax M3: Dual-Model Orchestration

Use your Claude Pro subscription's Opus for what it's best at — planning, architecture,
judgment calls — and offload the expensive part (repetitive tool-call loops: read file,
edit, run test, fix, repeat) to a MiniMax M3 model that speaks the Anthropic Messages API
natively. The result: your Anthropic 5-hour quota barely moves, because the iteration-heavy
work never touches it.

```
Main session (Claude Pro, manually switched to Opus)
  └─ Only responsible for: understand the task → break it down → write an executable plan
       ↓
claude-m3 (a thin CLI wrapper pointed at MiniMax M3, doesn't touch Anthropic's quota)
  └─ Responsible for: repeatedly execute the plan → read files, edit code, run tests, fix bugs
```

## Why this works

Claude Code only ever speaks one wire protocol: Anthropic's `/v1/messages` API. Normally,
running Claude Code against a non-Anthropic model requires a translation gateway (e.g. vLLM
with `--enable-auto-tool-choice`, or a hand-rolled adapter). **MiniMax ships an official
Anthropic-compatible endpoint**, so that translation layer is unnecessary here — you point
`ANTHROPIC_BASE_URL` straight at MiniMax and it just works, including tool calls.

## Quick start

1. Get a MiniMax API key.
2. Create `~/.claude/settings-mmx.json`:

   ```json
   {
     "env": {
       "ANTHROPIC_BASE_URL": "https://api.minimaxi.com/anthropic",
       "ANTHROPIC_AUTH_TOKEN": "<your-minimax-api-key>",
       "ANTHROPIC_DEFAULT_OPUS_MODEL": "MiniMax-M3",
       "ANTHROPIC_DEFAULT_SONNET_MODEL": "MiniMax-M2.7-highspeed",
       "ANTHROPIC_DEFAULT_HAIKU_MODEL": "MiniMax-M2.7-highspeed"
     }
   }
   ```

3. Create an executable wrapper at `~/bin/claude-m3` (**not** a shell alias — Claude Code's
   Bash tool runs a non-interactive, non-login shell that never sources `.zshrc`, so aliases
   silently don't exist):

   ```bash
   #!/bin/bash
   exec claude --settings ~/.claude/settings-mmx.json "$@"
   ```

   ```bash
   chmod +x ~/bin/claude-m3
   ```

4. Test text dialogue, then tool calling:

   ```bash
   claude-m3 -p "introduce yourself in one sentence" --output-format json

   claude-m3 -p "create test.txt in the current dir with content hello" \
     --allowedTools "Write" --output-format json
   ```

5. **Critical gotcha**: always pass `--allowedTools` in non-interactive (`-p`) mode. Without
   it, any tool call blocks on a permission prompt that a non-interactive session can never
   answer, and the process hangs.

6. Add a delegation rule to your project's `CLAUDE.md` so your main session knows when to
   hand off to `claude-m3` instead of doing the work itself. See [`CLAUDE.md`](./CLAUDE.md)
   in this repo for the exact rule used here.

## Verified results

| Test | Result |
|---|---|
| Text dialogue | ✅ Works. First call ttft ~176s (cold start, unexplained), later calls ~7–12s |
| Tool calling (`Write`) | ✅ Works reliably, no permission-prompt hang, 12s |
| End-to-end bug-fix task (delegate → execute → independently re-verify) | ✅ 5 tool-call turns, 25s, $0.34, fix confirmed correct by re-running the test independently rather than trusting the model's self-report |

## Repo structure — this is also a living knowledge base

This repo doubles as a knowledge base following Andrej Karpathy's ["LLM Wiki" pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f):
raw material is immutable, an LLM maintains the wiki on top of it, and a single `CLAUDE.md`
is the schema that governs how.

- `raw/` — immutable source material: the original design doc and the full session
  transcript that produced this repo
- `wiki/` — LLM-maintained knowledge base (topic pages, `INDEX.md`, `log.md`). **Written in
  Chinese** — start at [`wiki/INDEX.md`](./wiki/INDEX.md) for the full narrative, including
  the test transcripts and open questions.
- `outputs/` — generated deliverables (currently empty)
- `CLAUDE.md` — the single rule file: both the wiki-maintenance schema and the actual
  execution-delegation rule used to hand work off to `claude-m3`

## Known limitations / open questions

- The original plan's "build your own translation gateway" steps were skipped entirely
  because MiniMax happens to offer an official Anthropic-compatible endpoint. If you're
  pairing Claude Code with a model that doesn't offer this, you still need a gateway (vLLM
  with tool-call support, or a custom adapter) — see
  [`wiki/MiniMax-M3协议兼容与网关.md`](./wiki/MiniMax-M3协议兼容与网关.md).
- The 176-second first-call latency is unexplained and only observed once.
- Only tested on a single-file bug-fix task — not yet stress-tested on multi-file changes,
  full test suites, or what happens when a delegated task fails or hangs. See
  [`wiki/待澄清与后续事项.md`](./wiki/待澄清与后续事项.md).
