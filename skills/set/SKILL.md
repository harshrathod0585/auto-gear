---
name: Set
description: Use when the user explicitly asks to set, change, view, or reset their model spending cap or reasoning-effort ceiling — phrases like "set model cap", "change my max model", "what's my model policy", "reset my cap", "auto-gear:set". This is the setup-only half of the auto-gear skill; for the routing/dispatch behavior that applies the cap, see the sibling `auto-gear` skill.
---

## Set

Interactively configures the cap used by the `auto-gear` skill. This skill only writes the config file — it does not itself decide which model to use for a task; that's `auto-gear`'s job.

### Config file

Policy lives at `~/.claude/model-policy.json`:

```json
{
  "max_model": "sonnet-5",
  "max_reasoning_effort_by_model": {
    "haiku-4-5": null,
    "sonnet-5": "high"
  }
}
```

`max_reasoning_effort_by_model` is a per-model ceiling, not one global value — different models can be capped differently. Only include entries for models within `max_model`'s allowed set; a model with no effort concept (e.g. haiku) gets `null`.

Don't guess model names, capability ranking, or reasoning-effort support from memory. Before asking the user anything, invoke the `claude-api` skill (bundled with Claude Code) to get the authoritative current model lineup, their relative capability, and which reasoning-effort levels each supports.

If `claude-api` doesn't cover something needed (e.g. a brand-new model not yet documented there), WebFetch Anthropic's official model documentation page as a fallback before asking the user to fill the gap themselves.

### Steps

1. Read `~/.claude/model-policy.json`. If it exists, show the user their current settings and ask if they want to change them.
2. If it doesn't exist, tell the user you're setting up their model cap for the first time. Invoke the `claude-api` skill to get the current model lineup, capability ranking, and reasoning-effort support (never guess this). Then ask via `AskUserQuestion`:
   - "What's the highest model you want subagents to use?" — options: the current model lineup you just determined
   - Once `max_model` is picked, ask a follow-up per model within the allowed set (models below the cap, one question each — batch up to 4 per `AskUserQuestion` call): "Max reasoning effort for `<model>`?" — options: that model's valid effort levels. Skip this for any model with no effort/thinking concept.
3. Write the answers to `~/.claude/model-policy.json` as `max_model` + `max_reasoning_effort_by_model`, creating the `~/.claude/` directory first if needed.
4. Confirm the saved policy back to the user in one line.

The user can rerun this anytime to change their mind — always overwrite, don't merge or ask "are you sure."
