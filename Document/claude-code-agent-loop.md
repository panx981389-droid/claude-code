# Claude Code — The Agent Loop & All Prompts

> A code-level walkthrough of Claude Code's main agent loop and a complete catalog of
> every prompt the agent uses, with links into the source. Prompt **text** is not copied
> here on purpose — follow the code links to read the live strings.
>
> Companion diagrams: [`claude-code-agent-loop.drawio`](./claude-code-agent-loop.drawio)
> (3 pages: *Agent Loop*, *System Prompt Assembly*, *Prompt Catalog Map*).

All links are relative to the repository root. A reference like `src/query.ts:392` means
file `src/query.ts`, line `392`.

---

## 1. The Big Picture

Each user message kicks off a **turn loop**. One trip through the loop = one model call +
(optionally) a batch of tool executions. The loop keeps going while the model keeps asking
for tools, and stops when the model produces a final answer with no tool calls (or hits an
abort/limit condition).

```
User prompt
   │
   ▼
QueryEngine.submitMessage()  ──►  query()  ──►  queryLoop()  ──┐
                                                              │  while(true)
   ┌──────────────────────────────────────────────────────────┘
   │
   ├─ (1) compaction cascade  (snip → microcompact → collapse → autocompact)
   ├─ (2) assemble system prompt + context + tools
   ├─ (3) callModel()  → stream assistant message, collect tool_use blocks
   ├─ (4) stop hooks
   ├─ (5) runTools()  → execute tool calls, gather tool_result blocks
   ├─ (6) attach (queued cmds, memory, skill/tool discovery)
   └─ (7) needsFollowUp? ── yes ─► continue (next turn)
                          └─ no ──► return Terminal{reason}
```

See **Page 1 – "Agent Loop"** in the drawio for the full flowchart.

---

## 2. The Major Agent Loop (code level)

### 2.1 Entry points

| Layer | Symbol | Location | Role |
|------|--------|----------|------|
| REPL/headless orchestrator | `QueryEngine.submitMessage()` | [`src/QueryEngine.ts:217`](../src/QueryEngine.ts#L217) | High-level driver: builds system prompt, tools, MCP, then drives `query()`. Manages compaction boundaries & file-history. |
| One-shot wrapper | `ask()` | [`src/QueryEngine.ts:1256`](../src/QueryEngine.ts#L1256) | Convenience generator that wires up a single ask. |
| Core generator | `query()` | [`src/query.ts:275`](../src/query.ts#L275) | Public async generator. Yields stream events, messages, tombstones; returns a `Terminal`. |
| The actual loop | `queryLoop()` | [`src/query.ts:392`](../src/query.ts#L392) | The `while(true)` engine described below. |

`query()` and `queryLoop()` are **async generators** — they `yield` streaming events and
messages to the caller (the REPL renders them), and `return` a `Terminal` value describing
why the loop ended.

### 2.2 The `while (true)` engine

The loop body lives in `queryLoop()`, starting at [`src/query.ts:459`](../src/query.ts#L459).
Each iteration is one **turn**:

**(1) Pre-call compaction cascade** — runs *before* every model call so the request fits the
context window. Four independent stages, each feature-gated:

| Stage | Location | Trigger |
|-------|----------|---------|
| Snip compact | [`src/query.ts:574`](../src/query.ts#L574) | `feature('HISTORY_SNIP')` |
| Microcompact | [`src/query.ts:586`](../src/query.ts#L586) | `feature('CACHED_MICROCOMPACT')` |
| Context collapse | [`src/query.ts:623`](../src/query.ts#L623) | `feature('CONTEXT_COLLAPSE')` |
| Autocompact | [`src/query.ts:636`](../src/query.ts#L636) | token threshold exceeded |
| Predictive autocompact | [`src/query.ts:837`](../src/query.ts#L837) | projected turn growth > threshold |

**(2) Assemble the request** — the full system prompt is finalized with system context at
[`src/query.ts:632`](../src/query.ts#L632) via `asSystemPrompt(appendSystemContext(...))`.
User context is prepended to the messages just before the call. See §3.

**(3) Model call & streaming** — the API call site:

```ts
for await (const message of deps.callModel({ messages, systemPrompt, tools, ... }))
```

at [`src/query.ts:884`](../src/query.ts#L884). As the stream arrives:
- assistant message blocks are yielded to the caller,
- `tool_use` blocks are collected into `toolUseBlocks[]` at [`src/query.ts:1064`](../src/query.ts#L1064), setting `needsFollowUp = true`,
- if streaming tool execution is enabled, tools begin running *during* the stream via `StreamingToolExecutor` ([`src/query.ts:1081`](../src/query.ts#L1081)).
- fallback model switch is handled on `FallbackTriggeredError` ([`src/query.ts:1137`](../src/query.ts#L1137)).

**(4) Stop hooks** — `handleStopHooks()` at [`src/query.ts:1542`](../src/query.ts#L1542) lets
configured hooks block the stop and inject follow-up work.

**(5) Tool execution** — when there are tool calls, results are produced either by draining
the streaming executor (`getRemainingResults()`, [`src/query.ts:1291`](../src/query.ts#L1291))
or by the batch orchestrator `runTools()` (imported at [`src/query.ts:110`](../src/query.ts#L110),
from [`src/services/tools/toolOrchestration.js`](../src/services/tools/toolOrchestration.ts)).
A per-turn tool-use summary is generated at [`src/query.ts:1689`](../src/query.ts#L1689).

**(6) Attachments for the next turn** — queued commands, extracted memory, and skill/tool
discovery results are appended as user-content attachments around
[`src/query.ts:1879`](../src/query.ts#L1879)–[`src/query.ts:1939`](../src/query.ts#L1939).

**(7) Continuation decision** — if `needsFollowUp` and `turnCount < maxTurns`, the loop
updates `state` ([`src/query.ts:2028`](../src/query.ts#L2028)) and `continue`s
([`src/query.ts:2041`](../src/query.ts#L2041)). Otherwise it `return`s a `Terminal`.

### 2.3 Loop state (carried between turns)

The per-iteration `State` type is defined at [`src/query.ts:259`](../src/query.ts#L259):
messages, `toolUseContext`, autocompact tracking, recovery counters, the pending tool-use
summary promise, `stopHookActive`, `turnCount`, and the `transition` reason.

### 2.4 Termination (`Terminal.reason`)

| Reason | Location |
|--------|----------|
| `completed` (final answer, no tools) | [`src/query.ts:1632`](../src/query.ts#L1632) |
| `aborted_streaming` | [`src/query.ts:1323`](../src/query.ts#L1323) |
| `aborted_tools` | [`src/query.ts:1794`](../src/query.ts#L1794) |
| `max_turns` | [`src/query.ts:2024`](../src/query.ts#L2024) |
| `prompt_too_long` | [`src/query.ts:1454`](../src/query.ts#L1454) |
| `model_error` | [`src/query.ts:1242`](../src/query.ts#L1242) |

---

## 3. System Prompt Assembly (per turn)

The request sent to the model is composed of three streams: **system prompt**,
**system context**, and **user context**. See **Page 2 – "System Prompt Assembly"**.

### 3.1 The assembly pipeline

| Step | Symbol | Location |
|------|--------|----------|
| Gather all three parts | `fetchSystemPromptParts()` | [`src/utils/queryContext.ts:44`](../src/utils/queryContext.ts#L44) |
| Build the default system prompt | `getSystemPrompt()` | [`src/constants/prompts.ts:409`](../src/constants/prompts.ts#L409) |
| Layer custom/memory/append on top | `submitMessage()` body | [`src/QueryEngine.ts:294`](../src/QueryEngine.ts#L294) |
| Build effective prompt (priority resolution) | `buildEffectiveSystemPrompt()` | [`src/utils/systemPrompt.ts:41`](../src/utils/systemPrompt.ts#L41) |
| Finalize with system context | `asSystemPrompt(appendSystemContext(...))` | [`src/query.ts:632`](../src/query.ts#L632) |

Priority order (highest wins): **override → coordinator → agent → custom → default**, with an
optional `appendSystemPrompt`, resolved in `buildEffectiveSystemPrompt()`.

### 3.2 `getSystemPrompt()` structure

`getSystemPrompt()` returns a `string[]` split by a cache boundary marker,
`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` ([`src/constants/prompts.ts:118`](../src/constants/prompts.ts#L118)).
Everything **before** it is static/cacheable; everything **after** is dynamic per session.

There are three code paths inside `getSystemPrompt()`:
1. **Simple** (`CLAUDE_CODE_SIMPLE`) — minimal identity prompt ([`src/constants/prompts.ts:415`](../src/constants/prompts.ts#L415)).
2. **Proactive/autonomous** — when proactive mode is active ([`src/constants/prompts.ts:431`](../src/constants/prompts.ts#L431)).
3. **Default** — the full prompt assembled from static + dynamic sections ([`src/constants/prompts.ts:456`](../src/constants/prompts.ts#L456)–[`src/constants/prompts.ts:529`](../src/constants/prompts.ts#L529)).

**Static sections** (cacheable, in order):

| Section | Symbol | Location |
|---------|--------|----------|
| Intro / identity / cyber-risk / URL safety | `getSimpleIntroSection()` | [`src/constants/prompts.ts:179`](../src/constants/prompts.ts#L179) |
| System (tool categories, permission modes, prompt-injection, hooks, compaction) | `getSimpleSystemSection()` | [`src/constants/prompts.ts:190`](../src/constants/prompts.ts#L190) |
| Doing tasks / code style / verification | `getSimpleDoingTasksSection()` | [`src/constants/prompts.ts:205`](../src/constants/prompts.ts#L205) |
| Actions (reversibility, destructive ops) | `getActionsSection()` | [`src/constants/prompts.ts:249`](../src/constants/prompts.ts#L249) |
| Using your tools | `getUsingYourToolsSection()` | [`src/constants/prompts.ts:263`](../src/constants/prompts.ts#L263) |
| Output efficiency / tone & style | `getOutputEfficiencySection()` | [`src/constants/prompts.ts:383`](../src/constants/prompts.ts#L383) |

**Dynamic sections** (registry-managed, cache-aware) — selected at
[`src/constants/prompts.ts:456`](../src/constants/prompts.ts#L456) and resolved by
`resolveSystemPromptSections()` ([`src/constants/systemPromptSections.ts:20`](../src/constants/systemPromptSections.ts#L20)):

| Section | Symbol | Location |
|---------|--------|----------|
| Session-specific guidance | `getSessionSpecificGuidanceSection()` | [`src/constants/prompts.ts:457`](../src/constants/prompts.ts#L457) |
| Memory (CLAUDE.md mechanics + MEMORY.md) | `loadMemoryPrompt()` / `buildMemoryPrompt()` | [`src/memdir/memdir.ts:272`](../src/memdir/memdir.ts#L272) |
| Ant model override | `getAntModelOverrideSection()` | [`src/constants/prompts.ts:140`](../src/constants/prompts.ts#L140) |
| Environment info | `computeSimpleEnvInfo()` / `computeEnvInfo()` | [`src/constants/prompts.ts:559`](../src/constants/prompts.ts#L559) |
| Language preference | `getLanguageSection()` | [`src/constants/prompts.ts:146`](../src/constants/prompts.ts#L146) |
| Output style | `getOutputStyleSection()` | [`src/constants/prompts.ts:155`](../src/constants/prompts.ts#L155) |
| MCP server instructions | `getMcpInstructionsSection()` / `getMcpInstructions()` | [`src/constants/prompts.ts:164`](../src/constants/prompts.ts#L164) / [`src/constants/prompts.ts:532`](../src/constants/prompts.ts#L532) |
| Scratchpad dir instructions | `getScratchpadInstructions()` | [`src/constants/prompts.ts:752`](../src/constants/prompts.ts#L752) |
| Function-result clearing | `getFunctionResultClearingSection()` | [`src/constants/prompts.ts:776`](../src/constants/prompts.ts#L776) |
| Summarize-tool-results | `SUMMARIZE_TOOL_RESULTS_SECTION` | [`src/constants/prompts.ts:796`](../src/constants/prompts.ts#L796) |
| Token budget | inline section | [`src/constants/prompts.ts:499`](../src/constants/prompts.ts#L499) |
| Brief (KAIROS) | `getBriefSection()` | [`src/constants/prompts.ts:798`](../src/constants/prompts.ts#L798) |
| Hooks | `getHooksSection()` | [`src/constants/prompts.ts:131`](../src/constants/prompts.ts#L131) |
| System reminders | `getSystemRemindersSection()` | [`src/constants/prompts.ts:135`](../src/constants/prompts.ts#L135) |
| Proactive/autonomous work | `getProactiveSection()` | [`src/constants/prompts.ts:815`](../src/constants/prompts.ts#L815) |

### 3.3 Identity strings

| Constant | Location | Used for |
|----------|----------|----------|
| `DEFAULT_PREFIX` | [`src/constants/system.ts:10`](../src/constants/system.ts#L10) | "You are Claude Code, Anthropic's official CLI for Claude." |
| `AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX` | [`src/constants/system.ts:13`](../src/constants/system.ts#L13) | Claude Code running within the Agent SDK |
| `AGENT_SDK_PREFIX` | [`src/constants/system.ts:16`](../src/constants/system.ts#L16) | Generic Claude agent on the Agent SDK |
| `CYBER_RISK_INSTRUCTION` | [`src/constants/cyberRiskInstruction.ts:24`](../src/constants/cyberRiskInstruction.ts#L24) | Security policy injected into intro |
| `DEFAULT_AGENT_PROMPT` | [`src/constants/prompts.ts:713`](../src/constants/prompts.ts#L713) | Default subagent identity |

### 3.4 System context & user context

| Part | Symbol | Location | Contents |
|------|--------|----------|----------|
| System context | `getSystemContext()` | [`src/context.ts:116`](../src/context.ts#L116) | git status (cache-broken) |
| Git status | `getGitStatus()` | [`src/context.ts:36`](../src/context.ts#L36) | branch, default branch, status, recent commits, user |
| User context | `getUserContext()` | [`src/context.ts:155`](../src/context.ts#L155) | CLAUDE.md files + current date |
| CLAUDE.md discovery | (claudemd) | [`src/utils/claudemd.ts`](../src/utils/claudemd.ts) | walks project hierarchy |

User context is **prepended to the messages** (not the system prompt) at the call site
([`src/query.ts:885`](../src/query.ts#L885), `prependUserContext(...)`).

---

## 4. Per-turn Prompt & Context Injection Summary

Within a single turn, these are the prompt-bearing inputs that reach the model:

1. **System prompt** — static identity + behavior sections, then dynamic per-session
   sections (§3.2). Cache split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`.
2. **System context** — appended to the system prompt (git status).
3. **User context** — prepended to message history (CLAUDE.md, date).
4. **Conversation messages** — full or post-compact-boundary history.
5. **Attachments** (between turns) — memory, skill/tool discovery, queued commands,
   `<system-reminder>` blocks ([`src/query.ts:1879`](../src/query.ts#L1879)+).
6. **Tool definitions** — each tool ships its own description prompt (§6).

---

## 5. Compaction / Summarization Prompts

When the conversation is compacted, a **separate model call** summarizes history. These
prompts force text-only output (no tools).

| Prompt | Symbol | Location |
|--------|--------|----------|
| No-tools preamble | `NO_TOOLS_PREAMBLE` | [`src/services/compact/prompt.ts:19`](../src/services/compact/prompt.ts#L19) |
| Full compaction summary | `BASE_COMPACT_PROMPT` | [`src/services/compact/prompt.ts:61`](../src/services/compact/prompt.ts#L61) |
| Partial (recent-only) | `PARTIAL_COMPACT_PROMPT` | [`src/services/compact/prompt.ts:145`](../src/services/compact/prompt.ts#L145) |
| Partial up-to (cache-friendly) | `PARTIAL_COMPACT_UP_TO_PROMPT` | [`src/services/compact/prompt.ts:208`](../src/services/compact/prompt.ts#L208) |
| Strip `<analysis>` / format | `formatCompactSummary()` | [`src/services/compact/prompt.ts:311`](../src/services/compact/prompt.ts#L311) |
| Wrap summary as user message | `getCompactUserSummaryMessage()` | [`src/services/compact/prompt.ts:337`](../src/services/compact/prompt.ts#L337) |

---

## 6. Tool Description Prompts

Every built-in tool exposes its own description prompt (the text the model sees in the tool
schema). They live under
[`packages/builtin-tools/src/tools/<Tool>/prompt.ts`](../packages/builtin-tools/src/tools).

| Tool | Symbol | Location |
|------|--------|----------|
| Bash | `getSimplePrompt()` | [`packages/builtin-tools/src/tools/BashTool/prompt.ts:275`](../packages/builtin-tools/src/tools/BashTool/prompt.ts#L275) |
| PowerShell | `getPrompt()` | [`packages/builtin-tools/src/tools/PowerShellTool/prompt.ts:73`](../packages/builtin-tools/src/tools/PowerShellTool/prompt.ts#L73) |
| FileEdit | `getEditToolDescription()` | [`packages/builtin-tools/src/tools/FileEditTool/prompt.ts:8`](../packages/builtin-tools/src/tools/FileEditTool/prompt.ts#L8) |
| FileRead | `renderPromptTemplate()` | [`packages/builtin-tools/src/tools/FileReadTool/prompt.ts:27`](../packages/builtin-tools/src/tools/FileReadTool/prompt.ts#L27) |
| FileWrite | `getWriteToolDescription()` | [`packages/builtin-tools/src/tools/FileWriteTool/prompt.ts:10`](../packages/builtin-tools/src/tools/FileWriteTool/prompt.ts#L10) |
| Glob | `DESCRIPTION` | [`packages/builtin-tools/src/tools/GlobTool/prompt.ts`](../packages/builtin-tools/src/tools/GlobTool/prompt.ts) |
| Grep | `getDescription()` | [`packages/builtin-tools/src/tools/GrepTool/prompt.ts:6`](../packages/builtin-tools/src/tools/GrepTool/prompt.ts#L6) |
| WebFetch | `DESCRIPTION` / `makeSecondaryModelPrompt()` | [`packages/builtin-tools/src/tools/WebFetchTool/prompt.ts:3`](../packages/builtin-tools/src/tools/WebFetchTool/prompt.ts#L3) |
| WebSearch | `getWebSearchPrompt()` | [`packages/builtin-tools/src/tools/WebSearchTool/prompt.ts:5`](../packages/builtin-tools/src/tools/WebSearchTool/prompt.ts#L5) |
| NotebookEdit | `PROMPT` | [`packages/builtin-tools/src/tools/NotebookEditTool/prompt.ts:3`](../packages/builtin-tools/src/tools/NotebookEditTool/prompt.ts#L3) |
| LSP | `DESCRIPTION` | [`packages/builtin-tools/src/tools/LSPTool/prompt.ts:3`](../packages/builtin-tools/src/tools/LSPTool/prompt.ts#L3) |
| Agent | `getPrompt()` | [`packages/builtin-tools/src/tools/AgentTool/prompt.ts:65`](../packages/builtin-tools/src/tools/AgentTool/prompt.ts#L65) |
| TaskCreate | `getPrompt()` | [`packages/builtin-tools/src/tools/TaskCreateTool/prompt.ts:5`](../packages/builtin-tools/src/tools/TaskCreateTool/prompt.ts#L5) |
| TaskUpdate | `PROMPT` | [`packages/builtin-tools/src/tools/TaskUpdateTool/prompt.ts:3`](../packages/builtin-tools/src/tools/TaskUpdateTool/prompt.ts#L3) |
| TaskList | `getPrompt()` | [`packages/builtin-tools/src/tools/TaskListTool/prompt.ts:5`](../packages/builtin-tools/src/tools/TaskListTool/prompt.ts#L5) |
| SendMessage | `getPrompt()` | [`packages/builtin-tools/src/tools/SendMessageTool/prompt.ts:5`](../packages/builtin-tools/src/tools/SendMessageTool/prompt.ts#L5) |
| AskUserQuestion | `ASK_USER_QUESTION_TOOL_PROMPT` | [`packages/builtin-tools/src/tools/AskUserQuestionTool/prompt.ts:32`](../packages/builtin-tools/src/tools/AskUserQuestionTool/prompt.ts#L32) |
| Brief | `BRIEF_TOOL_PROMPT` / `BRIEF_PROACTIVE_SECTION` | [`packages/builtin-tools/src/tools/BriefTool/prompt.ts:6`](../packages/builtin-tools/src/tools/BriefTool/prompt.ts#L6) |
| Skill | `getPrompt()` | [`packages/builtin-tools/src/tools/SkillTool/prompt.ts:174`](../packages/builtin-tools/src/tools/SkillTool/prompt.ts#L174) |
| ExecuteExtraTool | `getPrompt()` | [`packages/builtin-tools/src/tools/ExecuteTool/prompt.ts:6`](../packages/builtin-tools/src/tools/ExecuteTool/prompt.ts#L6) |

Other tool prompts (ConfigTool, DiscoverSkillsTool, SearchExtraToolsTool, SleepTool,
SnipTool, ListMcpResourcesTool, ReadMcpResourceTool, ScheduleCronTool, VaultHttpFetchTool,
MCPTool, RemoteTriggerTool, EnterWorktreeTool) follow the same `<Tool>/prompt.ts` pattern.

---

## 7. Sub-agent System Prompts & the Sub-agent Loop

The `Agent`/`Task` tools spawn **child agent loops** that reuse the same `query()` engine
but with a different *system prompt* selected by `agentType`. The agent runner builds the
default Claude-Code prompt then swaps in the agent-specific prompt:
- default prompt fetched at [`packages/builtin-tools/src/tools/AgentTool/AgentTool.tsx:636`](../packages/builtin-tools/src/tools/AgentTool/AgentTool.tsx#L636),
- agent-specific prompt at [`packages/builtin-tools/src/tools/AgentTool/AgentTool.tsx:658`](../packages/builtin-tools/src/tools/AgentTool/AgentTool.tsx#L658),
- subagent run loop at [`packages/builtin-tools/src/tools/AgentTool/runAgent.ts:901`](../packages/builtin-tools/src/tools/AgentTool/runAgent.ts#L901).

Built-in agent prompts:

| Agent type | Symbol | Location |
|-----------|--------|----------|
| general-purpose | `getGeneralPurposeSystemPrompt()` | [`packages/builtin-tools/src/tools/AgentTool/built-in/generalPurposeAgent.ts:19`](../packages/builtin-tools/src/tools/AgentTool/built-in/generalPurposeAgent.ts#L19) |
| Explore | `getExploreSystemPrompt()` | [`packages/builtin-tools/src/tools/AgentTool/built-in/exploreAgent.ts:13`](../packages/builtin-tools/src/tools/AgentTool/built-in/exploreAgent.ts#L13) |
| Plan | `getPlanV2SystemPrompt()` | [`packages/builtin-tools/src/tools/AgentTool/built-in/planAgent.ts:14`](../packages/builtin-tools/src/tools/AgentTool/built-in/planAgent.ts#L14) |
| Verification | `VERIFICATION_SYSTEM_PROMPT` | [`packages/builtin-tools/src/tools/AgentTool/built-in/verificationAgent.ts:10`](../packages/builtin-tools/src/tools/AgentTool/built-in/verificationAgent.ts#L10) |
| claude-code-guide | `getClaudeCodeGuideBasePrompt()` | [`packages/builtin-tools/src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts:23`](../packages/builtin-tools/src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts#L23) |
| statusline-setup | (built-in) | [`packages/builtin-tools/src/tools/AgentTool/built-in/statuslineSetup.ts`](../packages/builtin-tools/src/tools/AgentTool/built-in/statuslineSetup.ts) |

Subagent prompts get an environment-detail wrapper via
`enhanceSystemPromptWithEnvDetails()` ([`src/constants/prompts.ts:715`](../src/constants/prompts.ts#L715)),
and may load persistent agent memory via `loadAgentMemoryPrompt()`
([`packages/builtin-tools/src/tools/AgentTool/agentMemory.ts:135`](../packages/builtin-tools/src/tools/AgentTool/agentMemory.ts#L135)).

### Plan mode prompts

| Prompt | Location |
|--------|----------|
| Enter plan mode (external) | [`packages/builtin-tools/src/tools/EnterPlanModeTool/prompt.ts:16`](../packages/builtin-tools/src/tools/EnterPlanModeTool/prompt.ts#L16) |
| Enter plan mode (ant) | [`packages/builtin-tools/src/tools/EnterPlanModeTool/prompt.ts:53`](../packages/builtin-tools/src/tools/EnterPlanModeTool/prompt.ts#L53) |
| Exit plan mode v2 | [`packages/builtin-tools/src/tools/ExitPlanModeTool/prompt.ts:6`](../packages/builtin-tools/src/tools/ExitPlanModeTool/prompt.ts#L6) |
| Verify plan execution | [`packages/builtin-tools/src/tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.ts:42`](../packages/builtin-tools/src/tools/VerifyPlanExecutionTool/VerifyPlanExecutionTool.ts#L42) |

---

## 8. Memory, Coordination & Other Specialized Prompts

| Purpose | Symbol | Location |
|---------|--------|----------|
| Memory mechanics block | `buildMemoryLines()` / `buildMemoryPrompt()` | [`src/memdir/memdir.ts:199`](../src/memdir/memdir.ts#L199) / [`src/memdir/memdir.ts:272`](../src/memdir/memdir.ts#L272) |
| Memory extraction (auto) | `buildExtractAutoOnlyPrompt()` | [`src/services/extractMemories/prompts.ts:50`](../src/services/extractMemories/prompts.ts#L50) |
| Memory extraction (combined) | `buildExtractCombinedPrompt()` | [`src/services/extractMemories/prompts.ts:101`](../src/services/extractMemories/prompts.ts#L101) |
| Cross-session recall | `PROMPT` | [`packages/builtin-tools/src/tools/LocalMemoryRecallTool/prompt.ts:10`](../packages/builtin-tools/src/tools/LocalMemoryRecallTool/prompt.ts#L10) |
| Session memory template | `DEFAULT_SESSION_MEMORY_TEMPLATE` | [`src/services/SessionMemory/prompts.ts:13`](../src/services/SessionMemory/prompts.ts#L13) |
| Session memory update | `buildSessionMemoryUpdatePrompt()` | [`src/services/SessionMemory/prompts.ts:226`](../src/services/SessionMemory/prompts.ts#L226) |
| Magic Docs update | `getUpdatePromptTemplate()` | [`src/services/MagicDocs/prompts.ts:10`](../src/services/MagicDocs/prompts.ts#L10) |
| Dream consolidation | `buildConsolidationPrompt()` | [`src/services/autoDream/consolidationPrompt.ts:10`](../src/services/autoDream/consolidationPrompt.ts#L10) |
| Away-summary recap (EN/ZH) | `RECAP_PROMPT_EN` / `RECAP_PROMPT_ZH` | [`src/commands/recap/generateRecap.ts:26`](../src/commands/recap/generateRecap.ts#L26) |
| Coordinator system prompt | `getCoordinatorSystemPrompt()` | [`src/coordinator/coordinatorMode.ts:111`](../src/coordinator/coordinatorMode.ts#L111) |
| Coordinator user context | `getCoordinatorUserContext()` | [`src/coordinator/coordinatorMode.ts:80`](../src/coordinator/coordinatorMode.ts#L80) |
| Teammate addendum | `TEAMMATE_SYSTEM_PROMPT_ADDENDUM` | [`src/utils/swarm/teammatePromptAddendum.ts:8`](../src/utils/swarm/teammatePromptAddendum.ts#L8) |
| Buddy/companion intro | `companionIntroText()` | [`src/buddy/prompt.ts:7`](../src/buddy/prompt.ts#L7) |
| Permission risk explainer | `SYSTEM_PROMPT` | [`src/utils/permissions/permissionExplainer.ts:44`](../src/utils/permissions/permissionExplainer.ts#L44) |
| Claude-in-Chrome base | `BASE_CHROME_PROMPT` | [`src/utils/claudeInChrome/prompt.ts:1`](../src/utils/claudeInChrome/prompt.ts#L1) |
| UltraPlan prompt selector | `getPromptText()` | [`src/utils/ultraplan/prompt.ts:6`](../src/utils/ultraplan/prompt.ts#L6) |

---

## 9. Diagrams

Open [`claude-code-agent-loop.drawio`](./claude-code-agent-loop.drawio) (draw.io / diagrams.net).
It contains three pages:

1. **Agent Loop** — the `queryLoop()` `while(true)` flow: compaction cascade → assemble →
   callModel → stop hooks → runTools → attach → continue/terminate, with line references.
2. **System Prompt Assembly** — how `fetchSystemPromptParts()` → `getSystemPrompt()` builds
   the static + dynamic sections around the cache boundary, plus system/user context.
3. **Prompt Catalog Map** — a map of every prompt family (core, tool, sub-agent, compaction,
   memory, specialized) and where it plugs into the loop.

---

## 10. Quick Reference — Key Files

| Concern | File |
|---------|------|
| Core loop | [`src/query.ts`](../src/query.ts) |
| Loop orchestrator | [`src/QueryEngine.ts`](../src/QueryEngine.ts) |
| System prompt content | [`src/constants/prompts.ts`](../src/constants/prompts.ts) |
| Prompt sections registry | [`src/constants/systemPromptSections.ts`](../src/constants/systemPromptSections.ts) |
| Identity prefixes | [`src/constants/system.ts`](../src/constants/system.ts) |
| Context (git/CLAUDE.md/date) | [`src/context.ts`](../src/context.ts) |
| Prompt-parts gatherer | [`src/utils/queryContext.ts`](../src/utils/queryContext.ts) |
| Effective prompt builder | [`src/utils/systemPrompt.ts`](../src/utils/systemPrompt.ts) |
| Compaction prompts | [`src/services/compact/prompt.ts`](../src/services/compact/prompt.ts) |
| Tool prompts | [`packages/builtin-tools/src/tools/*/prompt.ts`](../packages/builtin-tools/src/tools) |
| Sub-agent prompts | [`packages/builtin-tools/src/tools/AgentTool/built-in/`](../packages/builtin-tools/src/tools/AgentTool/built-in) |
