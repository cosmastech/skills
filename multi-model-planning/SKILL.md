---
name: multi-model-planning
description: >-
  Plan complex refactors and architectural changes by drafting a proposal and
  iterating through review rounds with multiple AI models via agentic CLIs
  (Cursor Agent, Claude Code, OpenAI Codex). Use when the user asks to plan
  a change with sub-agents, run a multi-model review, get consensus from
  different models, or mentions "planning session," "sub-agent review," or
  "iterate with models."
compatibility: >-
  Requires at least one agentic CLI that can send prompts to models and
  print responses. Supported CLIs: Cursor Agent (`agent`, install via
  `curl https://cursor.com/install -fsS | bash`), Claude Code (`claude`),
  OpenAI Codex (`codex`). Requires Shell tool access.
---

# Multi-Model Planning

Drive complex decisions to consensus by drafting a plan, sending it to multiple
AI models in parallel, synthesizing feedback, and iterating until agreement.

## Prerequisites

At least one agentic CLI must be installed and authenticated:

| CLI | Verify | Print-mode flag |
|-----|--------|-----------------|
| Cursor Agent | `agent --version` | `agent --model <model> --print "<prompt>"` |
| Claude Code | `claude --version` | `claude --model <model> -p "<prompt>"` |
| OpenAI Codex | `codex --version` | `codex --model <model> -q "<prompt>"` |

If cursor's `agent` is installed, always prefer using that and specifying models rather than mixing and matching CLI utilities.

## Workflow

### 1. Draft the plan

Create a branch-scoped working directory so artifacts from different planning
sessions never collide. Track the revision number starting at 1:

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PLAN_DIR="/tmp/planning-${BRANCH}"
mkdir -p "$PLAN_DIR"
REVISION=1
```

Write the proposal using the template in
[assets/planning-prompt-template.md](assets/planning-prompt-template.md) as a
starting point. The plan must include:

- **Context**: what exists today and why it needs to change.
- **Proposal**: the concrete change being considered.
- **Numbered questions**: specific decisions you need reviewer input on.

Write the file to `$PLAN_DIR/plan-r${REVISION}.md`.

Then build the full reviewer prompt by combining the reviewer instructions
template with the plan content:

```bash
cat assets/reviewer-prompt-template.md "$PLAN_DIR/plan-r${REVISION}.md" \
  > "$PLAN_DIR/plan-r${REVISION}-review.md"
```

See [assets/reviewer-prompt-template.md](assets/reviewer-prompt-template.md)
for the reviewer role, instructions, and expected output format.

### 2. Send to models in parallel

Fire both review requests in a single message so they run concurrently.
Use whichever CLI(s) and models the user prefers. If the user does not
specify, you always want to use the latest model that is high thinking
for the providers.

Do not assume the models that are available. Always check with the CLI tool first.
If you are unclear, ask the user which model they would like to use. Provide your
suggestions first.

Examples:

Capture each model's output into `$PLAN_DIR/` alongside the prompt that
produced it:

```bash
# Cursor Agent
agent --model gpt-5.4-xhigh --print "$(cat "$PLAN_DIR/plan-r${REVISION}-review.md")" 2>&1 | tee "$PLAN_DIR/plan-r${REVISION}-response-gpt.md"
agent --model claude-opus-4-7-thinking-xhigh --print "$(cat "$PLAN_DIR/plan-r${REVISION}-review.md")" 2>&1 | tee "$PLAN_DIR/plan-r${REVISION}-response-opus.md"

# Claude Code
claude --model opus -p "$(cat "$PLAN_DIR/plan-r${REVISION}-review.md")" 2>&1 | tee "$PLAN_DIR/plan-r${REVISION}-response-opus.md"

# OpenAI Codex
codex --model o3 -q "$(cat "$PLAN_DIR/plan-r${REVISION}-review.md")" 2>&1 | tee "$PLAN_DIR/plan-r${REVISION}-response-o3.md"
```

If you are using `agent`, you can select different models. This is preferred.  Otherwise attempt to mix and match CLIs to get diverse perspectives across providers. If there is only one CLI, you can still practice this exercise, you just won't have the diversity of models.

### 3. Synthesize feedback

After both responses return:

1. List points of **agreement** (these are decided).
2. List points of **disagreement** with each model's position.
3. Identify **critical issues** raised by either model (behavior changes,
   missing edge cases, sequencing gotchas).
4. Summarize the synthesis to the user before proceeding.

### 4. Draft round N+1

Bump the revision number and write the consolidated plan:

```bash
REVISION=$((REVISION + 1))
```

Write the updated plan to `$PLAN_DIR/plan-r${REVISION}.md`, incorporating
round N feedback. Include:

- A **decision log** for each resolved question.
- Any **new questions** surfaced by reviewers.
- Updated code sketches if the design changed.

Build the reviewer prompt for this round:

```bash
cat assets/reviewer-prompt-template.md "$PLAN_DIR/plan-r${REVISION}.md" \
  > "$PLAN_DIR/plan-r${REVISION}-review.md"
```

Send the updated plan to both models again. Repeat until:

- Both models agree the plan is sound.
- No unresolved critical issues remain.

Previous revisions are preserved in `$PLAN_DIR/` for reference.

### 5. Execute

Once consensus is reached, proceed with implementation. Mark the planning
todo items as completed and begin the implementation todos.

## Guidelines

- **Always use thinking models.** Planning reviews require deep reasoning.
  Never use non-thinking models (e.g. `gpt-4o`, `claude-sonnet`) for plan
  review. Use models with extended thinking capabilities (e.g.
  `gpt-5.3-codex-high`, `composer-2`, `claude-sonnet-4-thinking`,
  `o3`). If unsure whether a model supports thinking, ask the user.
- **Use different providers.** Ideally each reviewer model should be from a
  different provider (e.g. one OpenAI, one Anthropic, one Cursor) to avoid
  correlated blind spots.
- **Send plans with no prior context.** Each reviewer must receive the plan
  cold — do not include your own analysis, conclusions, or opinions in the
  prompt. The reviewer should form an independent assessment.
- **Challenge feedback that is wrong.** Consensus does not mean capitulation.
  If a reviewer's recommendation is based on a misunderstanding of the
  codebase, an incorrect assumption, or conflicts with established project
  conventions, push back with evidence. Explain why they're wrong and send
  the correction in the next round.
- **Never skip iteration.** If a model raises a critical issue, address it in
  a new round even if the other model approved.
- **Preserve behavior.** If a reviewer flags that the plan introduces a
  behavior change, explicitly confirm whether that change is intentional before
  proceeding.
- **Keep prompts self-contained.** Each round's prompt must include enough
  context for the model to review without access to prior rounds. Models do
  not share memory across invocations.
- **Show your work.** Summarize the synthesis to the user between rounds so
  they can course-correct.
- **Use todos.** Create a todo list that tracks: plan drafting, review rounds,
  implementation, testing, and pipeline runs.

## Typical todo structure

```
- Draft plan for <change>                         [in_progress]
- Run plan through models, iterate to consensus   [pending]
- Implement the agreed change                     [pending]
- Write tests for new code                        [pending]
- Run quality pipeline + full test suite           [pending]
- Push + start CI pipelines                       [pending]
```
