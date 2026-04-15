# tmux-agent-supervisor

A Claude Code skill that lets Claude supervise any long-running AI coding agent running in a tmux session — with automatic session discovery, AGENTS.md state management, and a 1-minute watchdog.

Works with any tmux-hosted agent: Codex CLI, Claude Code, Aider, etc.

> **This skill only works with [Claude Code](https://claude.ai/code).** It uses Claude Code-specific tools (CronCreate, tmux Bash access) that are not available in other Claude interfaces.

## What it does

- Explores your tmux sessions and summarizes what each agent is doing
- Establishes an `AGENTS.md` contract so task state is always recoverable
- Runs a 1-minute watchdog that detects RUNNING / IDLE / BLOCKED / NEED_DECISION / DRIFTING / BROKEN / DONE states
- Intervenes autonomously for routine situations; only escalates to you for real decisions

## Install

Paste this prompt into Claude Code:

```
请从 https://raw.githubusercontent.com/kongjiadongyuan/tmux-agent-supervisor/main/skills/tmux-agent-supervisor/SKILL.md 下载内容，保存到 ~/.claude/skills/tmux-agent-supervisor/SKILL.md（目录不存在则创建），完成后告诉我。
```

Then **restart Claude Code** to load the skill.

## Usage

Just describe what you want in natural language:

- "帮我看着这个 agent"
- "接管正在跑的 codex 会话"
- "开一个新的 agent 会话，做 X 任务"
