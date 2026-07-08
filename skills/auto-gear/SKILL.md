---
name: Auto Gear
description: Use when about to dispatch a subagent (Agent tool call) and need to pick which model to run it on, or when the user asks to set/change/view their model spending cap or preferred max model. Also use at the start of a session if no cap has ever been configured, to do first-run setup. Triggers on phrases like "set model cap", "change my max model", "what's my model policy", "don't use opus", "cap the model", "auto gear", or any request to control which Claude model gets used for tasks.
---

## Auto Gear

Lets the user set a ceiling on which Claude model gets used for subagent work, and gives a routing heuristic so the ceiling is respected automatically instead of the assistant picking models by feel.

**Why this exists:** a static model wastes money on trivial subagent tasks (haiku would do) and underperforms on hard ones (needs a stronger model). This skill makes model choice a policy the user controls, not a guess made silently.

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

`max_reasoning_effort_by_model` is a per-model ceiling, not one global value — different models can be capped differently (e.g. let sonnet-5 run up to xhigh but keep a pricier tier capped at low to control cost). Only include entries for models within `max_model`'s allowed set; a model with no effort concept (e.g. haiku) gets `null`.

Don't guess model names, capability ranking, or reasoning-effort support from memory — that produces wrong answers (e.g. mis-ranking a model that's actually strongest). Before asking the user anything, invoke the `claude-api` skill (bundled with Claude Code) to get the authoritative current model lineup, their relative capability, and which reasoning-effort levels each supports. That skill is maintained alongside Claude Code itself, so it's far more reliable than guessing from training knowledge or from name patterns (numbers, "mini"/"pro" suffixes don't indicate rank).

If `claude-api` doesn't cover something needed (e.g. a brand-new model not yet documented there), WebFetch Anthropic's official model documentation page as a fallback before asking the user to fill the gap themselves.

### Setup / editing the cap

Run this whenever the user asks to set, change, or check their cap — or when about to route a task and the config file doesn't exist yet (first-run).

1. Read `~/.claude/model-policy.json`. If it exists, show the user their current settings and ask if they want to change them.
2. If it doesn't exist, tell the user you're setting up their model cap for the first time. Invoke the `claude-api` skill to get the current model lineup, capability ranking, and reasoning-effort support (see Config file section above — never guess this). Then ask via `AskUserQuestion`:
   - "What's the highest model you want subagents to use?" — options: the current model lineup you just determined
   - Once `max_model` is picked, ask a follow-up per model within the allowed set (models below the cap, one question each — batch up to 4 per `AskUserQuestion` call): "Max reasoning effort for `<model>`?" — options: that model's valid effort levels. Skip this for any model with no effort/thinking concept.
3. Write the answers to `~/.claude/model-policy.json` as `max_model` + `max_reasoning_effort_by_model`, creating the `~/.claude/` directory first if needed.
4. Confirm the saved policy back to the user in one line.

The user can rerun this anytime to change their mind — always overwrite, don't merge or ask "are you sure."

### Routing a task (applying the cap)

Before any `Agent` tool call, do this:

1. Read `~/.claude/model-policy.json`. If missing, run first-run setup above before proceeding.
2. Classify the task about to be delegated:
   - **cheapest-tier**: single lookup, listing files, simple formatting, mechanical find/replace, summarizing a short known file — no judgment calls.
   - **mid-tier**: normal coding — implementing a feature, fixing a bug, writing tests, most everyday dev work.
   - **top-tier**: deep multi-file architecture reasoning, ambiguous/underspecified problems, high-stakes correctness (security, data migration), or anything where getting it wrong is expensive to unwind.
3. Pick the tier the task needs, then clamp it down to the user's `max_model` if it exceeds the cap — never clamp upward. Example: task needs the strongest model, cap is a mid-tier model → use the mid-tier model and mention to the user that the task may have benefited from a stronger model but was capped.
4. Pass the resulting tier as the `model` param on the `Agent` call. Look up that tier's entry in `max_reasoning_effort_by_model` and clamp the reasoning-effort setting (for any subagent definition that exposes one) to that value — never clamp upward, and never borrow another model's cap.

### Quick reference

Cap allows every model at or below the chosen tier, in the current strength ordering (weakest → strongest) determined at setup time via the `claude-api` skill — this ordering changes as Anthropic ships new models, so it is never hardcoded here.

### Note on "automatic" enforcement

This skill governs the assistant's choices when it dispatches subagents — it can't force a hard block the way a system permission would. For a guaranteed hard cap that can't be talked around, add a `PreToolUse` hook on the `Agent` tool (not included in this skill).
