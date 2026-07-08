---
name: Auto Gear
description: Use when about to dispatch a subagent (Agent tool call) and need to pick which model to run it on, respecting the user's configured model cap. Triggers whenever spawning an Agent-tool subagent for any task. For setting/changing/viewing the cap itself, use the sibling `auto-gear:set` skill instead.
---

## Auto Gear

Routes subagent dispatch to the cheapest model tier that can handle the task, never exceeding the user's configured cap. This is the routing half of the auto-gear skill pair — for setup, see `auto-gear:set`.

**Why this exists:** a static model wastes money on trivial subagent tasks (haiku would do) and underperforms on hard ones (needs a stronger model). This skill makes model choice a policy the user controls, not a guess made silently.

### Config file

Reads policy from `~/.claude/model-policy.json` (written by `auto-gear:set`):

```json
{
  "max_model": "sonnet-5",
  "max_reasoning_effort_by_model": {
    "haiku-4-5": null,
    "sonnet-5": "high"
  }
}
```

If this file doesn't exist yet, invoke the `auto-gear:set` skill to run first-run setup before routing anything.

### Routing a task (applying the cap)

Before any `Agent` tool call, do this:

1. Read `~/.claude/model-policy.json`. If missing, invoke `auto-gear:set` first.
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
