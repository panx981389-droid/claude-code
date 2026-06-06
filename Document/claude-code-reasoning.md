# Claude Code — The Reasoning System

> How Claude Code reasons: native extended thinking, the **interleaved-thinking loop** (the
> actual reasoning loop), and the prompt-driven reasoning layered on top. Code links throughout,
> and the reasoning-bearing **prompt text quoted per step**.
>
> Companion diagram: [`claude-code-reasoning.drawio`](./claude-code-reasoning.drawio)
> (4 pages: *Reasoning Layers*, *Interleaved Thinking Loop*, *Trigger & Config Resolution*,
> *Macro Reasoning Loops*).
>
> Siblings: [`claude-code-agent-loop.md`](./claude-code-agent-loop.md),
> [`claude-code-system-prompt.md`](./claude-code-system-prompt.md),
> [`claude-code-skill-system.md`](./claude-code-skill-system.md).

All links are relative to the repo root. `src/foo.ts:42` = file `src/foo.ts`, line `42`.

---

## 1. Three Layers of Reasoning

Claude Code's "reasoning" is not one thing. It's three cooperating layers:

```
┌─ Layer 1 · NATIVE EXTENDED THINKING ───────────────────────────────┐
│   The model's own chain-of-thought. Configured per request via      │
│   ThinkingConfig (adaptive | enabled | disabled) → API `thinking`.  │
│   Emitted as `thinking` / `redacted_thinking` content blocks.       │
└─────────────────────────────────────────────────────────────────────┘
            │ thinking blocks are kept alive across tool calls
            ▼
┌─ Layer 2 · THE INTERLEAVED-THINKING LOOP (the reasoning loop) ─────┐
│   think → tool_use → tool_result → think → …  within ONE turn.      │
│   The "rules of thinking" preserve thinking across the trajectory   │
│   so the model reasons, acts, observes, and reasons again.          │
└─────────────────────────────────────────────────────────────────────┘
            │ all of it runs under a system prompt + tool prompts that
            ▼
┌─ Layer 3 · PROMPT-DRIVEN REASONING ───────────────────────────────┐
│   Reasoning behaviors written into prompts: working-memory notes,   │
│   verify-before-complete, plan mode (explore→design→verify),        │
│   ultraplan strategies, adversarial verification, compaction        │
│   <analysis> scratchpad.                                            │
└─────────────────────────────────────────────────────────────────────┘
```

Layer 1 is the *engine*, Layer 2 is the *loop*, Layer 3 is the *policy*. The sections below take
them in order.

---

## 2. Layer 1 — Native Extended Thinking (the engine)

### 2.1 `ThinkingConfig`

The reasoning mode for a request is one of three shapes
([`src/utils/thinking.ts:11`](../src/utils/thinking.ts#L11)):

```ts
export type ThinkingConfig =
  | { type: 'adaptive' }                       // model decides how much to think
  | { type: 'enabled'; budgetTokens: number }  // fixed thinking budget (older models)
  | { type: 'disabled' }                        // no thinking
```

- **Default selection** at session start: `shouldEnableThinkingByDefault()`
  ([`src/utils/thinking.ts:151`](../src/utils/thinking.ts#L151)) →
  `{ type: 'adaptive' }` when enabled, else `{ type: 'disabled' }`. Wired in QueryEngine init
  ([`src/QueryEngine.ts:288`](../src/QueryEngine.ts#L288)).
- **Model gates**: `modelSupportsThinking()` ([`thinking.ts:91`](../src/utils/thinking.ts#L91))
  and `modelSupportsAdaptiveThinking()` ([`thinking.ts:114`](../src/utils/thinking.ts#L114)) —
  adaptive is Opus 4.6+/4.7+ and Sonnet 4.6+; budget mode is the fallback for older Claude 4 models.
- **Budget**: `getMaxThinkingTokensForModel()`
  ([`src/utils/context.ts:233`](../src/utils/context.ts#L233)) = model output upper-limit − 1.

### 2.2 Turning config into the API `thinking` param

The adaptive-vs-budget decision happens when the request is built
([`src/services/api/claude.ts:1663`](../src/services/api/claude.ts#L1663)): if thinking is
enabled and the model supports adaptive, send `{ type: 'adaptive' }` (no budget); otherwise send
`{ type: 'enabled', budget_tokens }` capped at `maxOutputTokens − 1`. The param is attached to
the stream request at [`claude.ts:1789`](../src/services/api/claude.ts#L1789).

### 2.3 Beta headers

Two headers govern reasoning behavior
([`src/constants/betas.ts:4`](../src/constants/betas.ts#L4)):

```ts
export const INTERLEAVED_THINKING_BETA_HEADER = 'interleaved-thinking-2025-05-14'
export const REDACT_THINKING_BETA_HEADER     = 'redact-thinking-2026-02-12'
```

`INTERLEAVED_THINKING_BETA_HEADER` is what lets thinking blocks persist *between* tool calls (the
Layer-2 loop). It's added when not disabled and the model supports it
([`src/utils/betas.ts:225`](../src/utils/betas.ts#L225)). `REDACT_THINKING_BETA_HEADER` is added
for interactive Haiku sessions that don't show thinking summaries
([`betas.ts:244`](../src/utils/betas.ts#L244)).

### 2.4 Thinking blocks in the stream

Reasoning arrives as content blocks alongside text/tool_use. The stream handler initializes a
`thinking` block with an empty `thinking` + `signature`
([`claude.ts:2111`](../src/services/api/claude.ts#L2111)) and accumulates `thinking_delta` /
`signature_delta` ([`claude.ts:2209`](../src/services/api/claude.ts#L2209)). A block is a
thinking block when `type === 'thinking' || type === 'redacted_thinking'`
(`isThinkingBlock`, [`src/utils/messages.ts:5204`](../src/utils/messages.ts#L5204)). The
`signature` cryptographically binds the thinking to the API key + model — which is why it must be
stripped on fallback (§3.3).

---

## 3. Layer 2 — The Reasoning Loop (interleaved thinking)

This is the heart of "the reasoning loop." It is the agent turn loop
([`claude-code-agent-loop.md`](./claude-code-agent-loop.md)) seen through the thinking lens:

```
   ┌──────────────── one assistant trajectory (a turn) ───────────────┐
   │  think ──► tool_use ──► (tool runs) ──► tool_result ──► think ──► │
   │   ▲                                                          │     │
   │   └──────────── thinking blocks preserved ◄──────────────────┘     │
   └───────────────────────────────────────────────────────────────────┘
        repeats until the model answers with no tool_use (turn ends)
```

### 3.1 The rules of thinking

Three invariants make the loop work
([`src/query.ts:180`](../src/query.ts#L180), quoted verbatim):

> 1. A message that contains a thinking or redacted_thinking block must be part of a query whose `max_thinking_length > 0`
> 2. A thinking block may not be the last message in a block
> 3. Thinking blocks must be preserved for the duration of an assistant trajectory (a single turn, or if that turn includes a tool_use block then also its subsequent tool_result and the following assistant message)

Rule 3 is the loop: within a turn, each reasoning step stays in context as the model calls a
tool, sees the result, and reasons about it — so reasoning compounds across actions instead of
restarting. The `interleaved-thinking` beta (§2.3) is what permits this on the API side.

### 3.2 Preserving thinking under compaction

When context is trimmed mid-session, thinking turns are deliberately kept. Microcompact uses the
`clear_thinking_20251015` edit strategy and keeps thinking turns when thinking is active and
redact-thinking is off ([`src/services/compact/apiMicrocompact.ts:77`](../src/services/compact/apiMicrocompact.ts#L77)),
optionally narrowing to the last thinking turn after a long idle cache miss.

### 3.3 Stripping thinking on model fallback

A thinking block's `signature` is bound to the model that produced it. On a fallback model
switch, `stripSignatureBlocks()` ([`src/utils/messages.ts:5506`](../src/utils/messages.ts#L5506))
removes those blocks before the retry ([`src/query.ts:1167`](../src/query.ts#L1167)) — otherwise
the new model rejects the foreign signatures.

---

## 4. Layer 3a — Triggers: when reasoning deepens (prompt per step)

### 4.1 Session-level config precedence

Set once at startup ([`src/main.tsx:2996`](../src/main.tsx#L2996)), highest wins:
`--thinking <adaptive|enabled|disabled>` → `--max-thinking-tokens N` → `MAX_THINKING_TOKENS`
env → settings `alwaysThinkingEnabled` → default `{ type: 'adaptive' }`. Stored in
`toolUseContext.options.thinkingConfig` and passed to every API call.

### 4.2 The `ultrathink` keyword (per-turn)

When the user writes **ultrathink** in their message, it raises reasoning effort for that turn.
Detection is `hasUltrathinkKeyword()` (regex `/\bultrathink\b/i`,
[`src/utils/thinking.ts:30`](../src/utils/thinking.ts#L30)), gated by `isUltrathinkEnabled()`
([`thinking.ts:20`](../src/utils/thinking.ts#L20)).

**Step 1 — build the attachment** ([`src/utils/attachments.ts:1500`](../src/utils/attachments.ts#L1500)):

```ts
function getUltrathinkEffortAttachment(input: string | null): Attachment[] {
  if (!isUltrathinkEnabled() || !input || !hasUltrathinkKeyword(input)) return []
  logEvent('tengu_ultrathink', {})
  return [{ type: 'ultrathink_effort', level: 'high' }]
}
```

**Step 2 — the actual prompt injected into the turn** — the attachment becomes a meta user
message wrapped in a `<system-reminder>`
([`src/utils/messages.ts:4589`](../src/utils/messages.ts#L4589)). The full text:

```text
The user has requested reasoning effort level: high. Apply this to the current turn.
```

So `ultrathink` does **not** set `ThinkingConfig` directly — it injects an instruction that tells
the model to reason harder this turn, and (where supported) bumps the separate **effort** axis.

### 4.3 Effort (a separate reasoning axis)

`effort ∈ {low, medium, high, xhigh, max}` is resolved by `resolveAppliedEffort()`
([`src/utils/effort.ts:178`](../src/utils/effort.ts#L178)) (env → app state → model default) and
sent as its own API parameter. Thinking controls *how many* reasoning tokens; effort controls the
model's reasoning *strategy/depth*. `ultrathink` nudges effort to `high`.

---

## 5. Layer 3b — Reasoning written into the system prompt (prompt per step)

These sections of the system prompt
([`src/constants/prompts.ts`](../src/constants/prompts.ts)) shape *how* the model reasons during
ordinary turns. Quoted verbatim:

**Working memory** — synthesize before context is cleared (`SUMMARIZE_TOOL_RESULTS_SECTION`,
[`prompts.ts:796`](../src/constants/prompts.ts#L796)):

> When working with tool results, write down any important information you might need later in your response, as the original tool result may be cleared later.

**Verify before completion** ([`prompts.ts:215`](../src/constants/prompts.ts#L215)):

> Before reporting a task complete, verify it actually works: run the test, execute the script, check the output. … If you can't verify (no test exists, can't run the code), say so explicitly rather than claiming success.

**Faithful reporting / epistemic honesty** ([`prompts.ts:237`](../src/constants/prompts.ts#L237)):

> Report outcomes faithfully: if tests fail, say so with the relevant output; if you did not run a verification step, say that rather than implying it succeeded. …

**Diagnose before retrying** ([`prompts.ts:232`](../src/constants/prompts.ts#L232)):

> If an approach fails, diagnose why before switching tactics—read the error, check your assumptions, try a focused fix. Don't retry the identical action blindly … Escalate to the user with AskUserQuestion only when you're genuinely stuck after investigation …

**Collaborative judgment** ([`prompts.ts:228`](../src/constants/prompts.ts#L228)):

> If you notice the user's request is based on a misconception, or spot a bug adjacent to what they asked about, say so. You're a collaborator, not just an executor …

**Autonomous reasoning** (proactive mode) — `getProactiveSection()`
([`prompts.ts:815`](../src/constants/prompts.ts#L815)) calibrates how decisively to act based on
whether the user is watching ("bias toward action" when away, "be more collaborative" when
focused).

---

## 6. Layer 3c — Macro Reasoning Loops (explore → plan → verify)

Beyond a single turn, Claude Code has explicit multi-step reasoning loops driven entirely by
prompts.

### 6.1 Plan mode — explore, then design

Entering plan mode returns this instruction block (the reasoning recipe) as the tool result
([`packages/builtin-tools/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts:108`](../packages/builtin-tools/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts#L108)), quoted:

```text
In plan mode, you should:
1. Thoroughly explore the codebase to understand existing patterns
2. Identify similar features and architectural approaches
3. Consider multiple approaches and their trade-offs
4. Use AskUserQuestion if you need to clarify the approach
5. Design a concrete implementation strategy
6. When ready, use ExitPlanMode to present your plan for approval

Remember: DO NOT write or edit any files yet. This is a read-only exploration and planning phase.
```

`ExitPlanMode` signals the plan is ready (not a question)
([`ExitPlanModeTool/prompt.ts:6`](../packages/builtin-tools/src/tools/ExitPlanModeTool/prompt.ts#L6));
`VerifyPlanExecution` closes the loop by checking execution against the stated plan
([`VerifyPlanExecutionTool/VerifyPlanExecutionTool.ts:42`](../packages/builtin-tools/src/tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.ts#L42)).
The read-only **Plan agent** runs the same explore→design discipline as a subagent
([`AgentTool/built-in/planAgent.ts:14`](../packages/builtin-tools/src/tools/AgentTool/built-in/planAgent.ts#L14)).

### 6.2 UltraPlan — selectable reasoning strategies

`getPromptText(id)` ([`src/utils/ultraplan/prompt.ts:30`](../src/utils/ultraplan/prompt.ts#L30))
returns one of three strategies (chosen by GrowthBook, default `simple_plan`):

- **`simple_plan`** — single-pass, grounded, no subagents
  ([`prompts/simple_plan.txt`](../src/utils/ultraplan/prompts/simple_plan.txt)):

  > Explore the codebase directly with Glob, Grep, and Read. Read the relevant code, understand how the pieces fit, look for existing functions and patterns you can reuse instead of proposing new ones … Do not spawn subagents.

- **`visual_plan`** — adds a diagram tier so structure is verifiable at a glance
  ([`prompts/visual_plan.txt`](../src/utils/ultraplan/prompts/visual_plan.txt)).

- **`three_subagents_with_critique`** — parallel multi-agent exploration + adversarial critique
  ([`prompts/three_subagents_with_critique.txt`](../src/utils/ultraplan/prompts/three_subagents_with_critique.txt)), quoted:

  > 1. Use the Task tool to spawn parallel agents to explore different aspects of the codebase simultaneously:
  >    - One agent to understand the relevant existing code and architecture
  >    - One agent to find all files that will need modification
  >    - One agent to identify potential risks, edge cases, and dependencies
  > 2. Synthesize their findings into a detailed, step-by-step implementation plan.
  > 3. Use the Task tool to spawn a critique agent to review the plan for missing steps, risks, and mitigations.
  > 4. Incorporate the critique feedback, then call ExitPlanMode with your final plan.

These map to increasing reasoning thoroughness: ground-truth single pass → + visual structure →
+ parallel exploration with self-critique.

### 6.3 The verification loop — adversarial self-checking

The verification agent inverts the usual bias: its job is to *break* the implementation, not
confirm it ([`AgentTool/built-in/verificationAgent.ts:10`](../packages/builtin-tools/src/tools/AgentTool/built-in/verificationAgent.ts#L10)), quoted:

> Your job is not to confirm the implementation works — it's to try to break it. … being seduced by the first 80% … Your entire value is in finding the last 20%.

It must run adversarial probes (concurrency, boundary values, idempotency, orphan ops) and cite
command output as evidence — code-reading doesn't count. The main agent runs this as a contract:
on non-trivial work, spawn the verifier, and on FAIL **fix → resume verifier → repeat until
PASS** ([`src/constants/prompts.ts:366`](../src/constants/prompts.ts#L366)) — an
implement→verify→fix→re-verify reasoning loop.

### 6.4 Compaction reasoning — the `<analysis>` scratchpad

Summarizing the conversation is itself a reasoning task. The compaction prompt forces a private
scratchpad: respond text-only with an `<analysis>` block (stripped before it reaches context)
then a `<summary>` block (`NO_TOOLS_PREAMBLE` [`src/services/compact/prompt.ts:19`](../src/services/compact/prompt.ts#L19),
`BASE_COMPACT_PROMPT` [`prompt.ts:61`](../src/services/compact/prompt.ts#L61)). The `<analysis>`
section makes the model reason chronologically through the whole conversation before producing a
9-section structured summary.

---

## 7. Quick Reference

| Concern | Symbol | Location |
|---------|--------|----------|
| Thinking config type | `ThinkingConfig` | [`src/utils/thinking.ts:11`](../src/utils/thinking.ts#L11) |
| Default on/off | `shouldEnableThinkingByDefault()` | [`src/utils/thinking.ts:151`](../src/utils/thinking.ts#L151) |
| Model gates | `modelSupportsThinking` / `…AdaptiveThinking` | [`thinking.ts:91`](../src/utils/thinking.ts#L91) / [`:114`](../src/utils/thinking.ts#L114) |
| Budget | `getMaxThinkingTokensForModel` | [`src/utils/context.ts:233`](../src/utils/context.ts#L233) |
| Config init | QueryEngine | [`src/QueryEngine.ts:288`](../src/QueryEngine.ts#L288) |
| API thinking decision | request build | [`src/services/api/claude.ts:1663`](../src/services/api/claude.ts#L1663) |
| Beta headers | `INTERLEAVED_/REDACT_THINKING_BETA_HEADER` | [`src/constants/betas.ts:4`](../src/constants/betas.ts#L4) |
| Header inclusion | — | [`src/utils/betas.ts:225`](../src/utils/betas.ts#L225) |
| Thinking blocks (stream) | `isThinkingBlock` / deltas | [`messages.ts:5204`](../src/utils/messages.ts#L5204) / [`claude.ts:2209`](../src/services/api/claude.ts#L2209) |
| Rules of thinking | comment block | [`src/query.ts:180`](../src/query.ts#L180) |
| Thinking under compaction | `clear_thinking_20251015` | [`apiMicrocompact.ts:77`](../src/services/compact/apiMicrocompact.ts#L77) |
| Strip on fallback | `stripSignatureBlocks` | [`messages.ts:5506`](../src/utils/messages.ts#L5506) |
| Session config precedence | — | [`src/main.tsx:2996`](../src/main.tsx#L2996) |
| ultrathink keyword | `hasUltrathinkKeyword` | [`src/utils/thinking.ts:30`](../src/utils/thinking.ts#L30) |
| ultrathink attachment | `getUltrathinkEffortAttachment` | [`src/utils/attachments.ts:1500`](../src/utils/attachments.ts#L1500) |
| ultrathink injected prompt | `ultrathink_effort` case | [`src/utils/messages.ts:4589`](../src/utils/messages.ts#L4589) |
| Effort resolution | `resolveAppliedEffort` | [`src/utils/effort.ts:178`](../src/utils/effort.ts#L178) |
| Reasoning prompt sections | various | [`src/constants/prompts.ts:796`](../src/constants/prompts.ts#L796), [`:215`](../src/constants/prompts.ts#L215), [`:228`](../src/constants/prompts.ts#L228), [`:232`](../src/constants/prompts.ts#L232), [`:237`](../src/constants/prompts.ts#L237) |
| Plan mode instructions | `mapToolResultToToolResultBlockParam` | [`EnterPlanModeTool.ts:108`](../packages/builtin-tools/src/tools/EnterPlanModeTool/EnterPlanModeTool.ts#L108) |
| UltraPlan strategies | `getPromptText` | [`src/utils/ultraplan/prompt.ts:30`](../src/utils/ultraplan/prompt.ts#L30) |
| Verification agent | `VERIFICATION_SYSTEM_PROMPT` | [`verificationAgent.ts:10`](../packages/builtin-tools/src/tools/AgentTool/built-in/verificationAgent.ts#L10) |
| Verify contract | system prompt | [`src/constants/prompts.ts:366`](../src/constants/prompts.ts#L366) |
| Compaction `<analysis>` | `NO_TOOLS_PREAMBLE` / `BASE_COMPACT_PROMPT` | [`compact/prompt.ts:19`](../src/services/compact/prompt.ts#L19) / [`:61`](../src/services/compact/prompt.ts#L61) |
