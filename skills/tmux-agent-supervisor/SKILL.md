---
name: tmux-agent-supervisor
description: Use when asked to supervise, babysit, take over, or start a long-running AI coding agent in tmux. Triggers: "帮我看着 agent", "督导任务", "接管 codex 会话", "开始 agent 任务", or any request to manage an agent session. Covers both new sessions and takeover of existing ones.
---

# tmux Agent Supervisor

## Overview

Claude acts as **Supervisor** for any AI coding agent running in a tmux pane. Manages session discovery, establishes an `AGENTS.md` state contract, runs a 1-minute CronCreate watchdog, and escalates to the user only when required.

Works with any agent: Codex CLI, Claude Code, Aider, etc.

**CRITICAL:** Never use SubAgent tool, TeamCreate, or `claude` CLI to start a new agent. Always use tmux.

## Roles

| Role | Does | Doesn't |
|------|------|---------|
| User | Goal, directional decisions | Execution details |
| Claude | Discover, watchdog, intervene, escalate | Write code, fix bugs |
| Agent | Execute, maintain AGENTS.md | Architecture decisions |

---

## Step 1: Session Discovery

**ALWAYS start here — even for "new session" requests. Never skip this step.**

### Explore all tmux sessions first

```bash
tmux list-sessions
tmux list-windows -t <session>
tmux capture-pane -p -t <session>:<window>                     # recent output per window
tmux display-message -p -t <target> '#{pane_current_path}'     # working directory
```

**Filter rule:** Skip any window with no AI agent activity (e.g., bare `$` prompt, vim, etc.). Do NOT show these to the user.

**Summarize each agent window** in one plain-language sentence (e.g., "正在重构认证模块，当前写测试").

### Present human-readable options

**Never show raw `session:window.pane` notation.** Always use plain summaries.

Example:
```
我发现以下 agent 会话：

● Session "work" / Tab 2 — Claude Code 正在重构认证模块，当前在写测试
● Session "proj2" / Tab 1 — Aider 在修 API bug，最近无输出（可能卡住）

+ 在当前 session 新建 tab
+ 新建 session
```

"新建" option is always present regardless of what was found.
If no agent windows found: "没找到在跑的 agent，要新建吗？"

### After user selects

**New tab:**
```bash
tmux new-window -t <session>   # tab inside existing session, NOT tmux new-session
```
Then `send-keys` to launch agent + initial task + AGENTS.md requirement.

**Takeover:**
1. `capture-pane` last 50 lines → understand current state
2. `tmux display-message -p -t <target> '#{pane_current_path}'` → working directory
3. Check `./AGENTS.md` in that directory
4. No AGENTS.md → insert requirement when output pauses (see Step 2)
5. Has AGENTS.md → proceed to watchdog (Step 3)

---

## Step 2: AGENTS.md Contract

**Every supervised agent MUST maintain AGENTS.md. This is non-negotiable.**

**Location:** `./AGENTS.md` in the agent's working directory (`pane_current_path`)

**Requirement text to inject via send-keys:**

```
请创建并维护 AGENTS.md，结构如下：

# 当前任务
- 目标：
- 阶段：

# 当前计划
- [ ] Step 1
- [ ] Step 2

# 历史决策
- 选了 X 而不是 Y，原因：

# 下一步候选
- 方向 A（理由）
- 方向 B（理由）

# 风险 / 阻塞
- 当前卡点

规则：每次阶段推进后必须更新；方向变化必须记录原因；仅凭此文件+代码必须能恢复全部上下文。
```

**When to send:**
- New session: include in initial task prompt (before agent starts)
- Takeover without AGENTS.md: send via `send-keys` when output stops for >30s

---

## Step 3: Start Watchdog

After session is ready, set up CronCreate:

```
CronCreate(
  cron: "* * * * *",
  prompt: "巡检 tmux agent 会话 <target-pane>：执行标准巡检流程（见 tmux-agent-supervisor skill）",
  recurring: true
)
```

---

## Step 4: Watchdog Inspection Cycle

On each cron trigger:

1. `tmux capture-pane -p -t <target> -S -80` → last 80 lines
2. Read `AGENTS.md` at `pane_current_path`
3. Determine state and act:

| State | Detection | Action |
|-------|-----------|--------|
| `RUNNING` | New output since last check | No intervention |
| `IDLE` | No new output × 2 consecutive checks (≈2 min) | `send-keys "继续"` |
| `BLOCKED` | Same error/attempt repeating ≥ 3 times | `send-keys "brainstorming 当前阻塞点，给出解决路径"` |
| `NEED_DECISION` | Agent output shows options or explicit question | Summarize, escalate to user |
| `DRIFTING` | Output diverges from AGENTS.md target | `send-keys "回到 AGENTS.md 核心目标，继续最关键路径"` |
| `BROKEN` | Crashed / garbled / unresponsive | `Ctrl+C` ×2 → `send-keys "resume"` → notify user if fails |
| `DONE` | AGENTS.md shows convergence / goal achieved | CronDelete, notify user |

### IDLE vs BLOCKED — Critical Distinction

**IDLE = silence.** No output at all for 2+ consecutive checks.
**BLOCKED = repeated failure.** There IS output, but it's the same error looping.

**If you see output (even error output), it is NOT IDLE.**

### BLOCKED handling — Never touch the code

When state is BLOCKED:
1. Send exactly: `send-keys "brainstorming 当前阻塞点，给出解决路径"` — then STOP
2. Do NOT run `git diff`, `grep`, or read source files
3. Do NOT fix the error yourself
4. Do NOT send Ctrl+C (agent is running, not stuck)
5. Wait for next watchdog cycle to assess if agent unblocked

The agent is the executor. You are the supervisor. Supervisors do not write code.

### DRIFTING escalation

If send-keys intervention fails twice, escalate to user.

---

## Step 5: Human Collaboration Boundary

**Escalate to user ONLY for:**
- `NEED_DECISION`: summarize agent's options clearly, ask user to choose
- `BROKEN` + recovery failed: report state, ask what to do
- `DONE`: stop cron, summarize what was accomplished
- `DRIFTING` × 2 interventions failed: explain the drift, ask for direction

**Never disturb user for:**
- Implementation choices, debugging, optimizations
- Standard RUNNING / IDLE / BLOCKED handling

---

## Stop Conditions

- AGENTS.md shows task converged
- User says stop
- `BROKEN` and unrecoverable

**On stop:** `CronDelete` to clear watchdog, send brief summary to user of what was accomplished.

---

## Common Mistakes

| Mistake | Correct Behavior |
|---------|-----------------|
| Using Agent tool / TeamCreate / `claude` CLI for new session | Always use `tmux new-window` |
| Showing all tmux windows to user | Only show windows with AI agent activity |
| Identifying repeated-error output as IDLE | IDLE = silence. Repeated errors = BLOCKED |
| Running Ctrl+C / git diff / grep when BLOCKED | Send `send-keys "brainstorming..."` and wait |
| Fixing agent's code directly | Never write code. Send intervention prompt only |
| Using `tmux new-session` instead of `tmux new-window` | `new-window` creates a tab in existing session |
| Forgetting to CronDelete on stop | Always clean up cron when supervision ends |
| Skipping AGENTS.md requirement | Every session needs the AGENTS.md contract |
