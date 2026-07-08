# auto-gear

A Claude Code skill that lets you set a ceiling on which Claude model gets used for subagent work, and routes tasks to the right tier automatically (cheap models for simple lookups, stronger models for hard problems) — never exceeding your configured cap.

## Install

```
/plugin marketplace add harshrathod0585/auto-gear
/plugin install auto-gear
```

## Usage

Say "set model cap" (or similar) to configure your ceiling. Claude will ask which model tier and reasoning-effort level you want as the max, then save it to `~/.claude/model-policy.json`. From then on, whenever Claude dispatches a subagent, it classifies the task's difficulty and picks the cheapest model tier that can handle it — clamped to your cap.

Rerun the setup phrase anytime to change your mind.

## Why

Running every subagent task on the most powerful model wastes money on trivial work; running everything on a cheap model underperforms on hard problems. This skill makes model selection an explicit, user-controlled policy instead of a silent guess.

## Note on enforcement

This skill guides Claude's own choices — it's not a hard technical block. For a guaranteed cap, pair it with a `PreToolUse` hook on the `Agent` tool.
