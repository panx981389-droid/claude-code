# Claude Code — What Gets Sent to the LLM When a Skill Is Invoked

> This traces a single `Skill` invocation end to end and shows the **actual message text**
> sent to the model at each step: the rendered `SKILL.md` body, how skill information is
> appended, and how scripts inside a skill get executed (two distinct mechanisms).
>
> Key code: `SkillTool.call` ([`packages/builtin-tools/src/tools/SkillTool/SkillTool.ts`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts)),
> `getMessagesForPromptSlashCommand` ([`src/utils/processUserInput/processSlashCommand.tsx:1081`](../src/utils/processUserInput/processSlashCommand.tsx#L1081)),
> `getPromptForCommand` ([`src/skills/loadSkillsDir.ts:344`](../src/skills/loadSkillsDir.ts#L344)),
> `executeShellCommandsInPrompt` ([`src/utils/promptShellExecution.ts:69`](../src/utils/promptShellExecution.ts#L69)).

Companion docs: [`claude-code-skill-system.md`](./claude-code-skill-system.md),
[`claude-code-system-prompt.md`](./claude-code-system-prompt.md).

---

## 0. A Worked Example

We'll use this on-disk skill at `~/.claude/skills/pdf-extract/SKILL.md`:

````markdown
---
name: pdf-extract
description: Extract and summarize text from a PDF file
allowed-tools: Bash, Read
arguments: file
---
# PDF Extract

You are extracting text from the PDF at `$file`.

Current time for the log header: !`date +%H:%M`

Steps:
1. Run the bundled extractor script:

   ```
   python ${CLAUDE_SKILL_DIR}/scripts/extract.py "$file"
   ```

2. Read the produced `out.txt` and summarize it in 3 bullets.

Session: ${CLAUDE_SESSION_ID}
````

The model invokes it with:

```json
{ "name": "Skill", "input": { "skill": "pdf-extract", "args": "report.pdf" } }
```

---

## 1. Step 1 — The model calls the `Skill` tool

The model issues a `tool_use` block. Nothing of the skill body is in context yet — only the
`Skill` tool's own description prompt told it skills exist
([`SkillTool/prompt.ts:174`](../packages/builtin-tools/src/tools/SkillTool/prompt.ts#L174)),
including this directive:

```text
- When a skill matches the user's request, this is a BLOCKING REQUIREMENT: invoke the relevant Skill tool BEFORE generating any other response about the task
- If you see a <command-name> tag in the current conversation turn, the skill has ALREADY been loaded - follow the instructions directly instead of calling this tool again
```

`SkillTool.call` then runs: `validateInput` → `checkPermissions` → locate command → (inline vs.
fork). For inline skills it calls
`processPromptSlashCommand("pdf-extract", "report.pdf", …)`
([`SkillTool.ts:642`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L642)).

---

## 2. Step 2 — Rendering the `SKILL.md` body (`getPromptForCommand`)

This is where skill information is **assembled into a prompt string**. Four transformations run,
in order ([`src/skills/loadSkillsDir.ts:344`](../src/skills/loadSkillsDir.ts#L344)):

### 2a. Prepend the base directory

```ts
finalContent = baseDir
  ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
  : markdownContent
```

So the body now starts with:

```text
Base directory for this skill: /Users/you/.claude/skills/pdf-extract

# PDF Extract
...
```

### 2b. Substitute arguments (`substituteArguments`, [`argumentSubstitution.ts:94`](../src/utils/argumentSubstitution.ts#L94))

Args string `"report.pdf"` is parsed into positional args. Replacements applied in order:

| Placeholder | Replaced with | Note |
|-------------|---------------|------|
| `$file` (named, from `arguments: file`) | `report.pdf` | named args map by position: `argumentNames[0]→arg[0]` |
| `$ARGUMENTS[0]`, `$ARGUMENTS[1]`… | indexed arg | |
| `$0`, `$1`… | shorthand indexed arg | |
| `$ARGUMENTS` | full args string `report.pdf` | |
| *(none of the above present)* | append `\n\nARGUMENTS: <args>` | only if args non-empty |

Our `$file` → `report.pdf`.

### 2c. Replace `${CLAUDE_SKILL_DIR}` and `${CLAUDE_SESSION_ID}`

- `${CLAUDE_SKILL_DIR}` → the skill's own dir (`baseDir`), backslashes normalized to `/` on Windows.
- `${CLAUDE_SESSION_ID}` → the current session id.

### 2d. Execute inline shell blocks **NOW** (load-time scripts)

`executeShellCommandsInPrompt` ([`promptShellExecution.ts:69`](../src/utils/promptShellExecution.ts#L69))
finds two syntaxes and **runs them at load time**, replacing each match with its stdout:

- inline:  `` !`command` ``  (regex `INLINE_PATTERN`)
- block:   ` ```! command ``` `  (regex `BLOCK_PATTERN`)

Each command goes through a real permission check and `BashTool.call` (or `PowerShellTool` if the
skill's frontmatter says `shell: powershell`). **MCP skills are exempt — their bodies are never
shell-executed** (`loadedFrom !== 'mcp'` guard). Our `` !`date +%H:%M` `` runs and is replaced by
e.g. `14:07`.

> Note: the triple-backtick block in our example is a plain ` ``` ` (no `!`), so it is **not**
> executed here — it stays as literal instructions for the model to run later (Step 5).

### Result of Step 2 — the fully rendered skill text

`getPromptForCommand` returns one text block. For our example it is:

```text
Base directory for this skill: /Users/you/.claude/skills/pdf-extract

# PDF Extract

You are extracting text from the PDF at `report.pdf`.

Current time for the log header: 14:07

Steps:
1. Run the bundled extractor script:

   ```
   python /Users/you/.claude/skills/pdf-extract/scripts/extract.py "report.pdf"
   ```

2. Read the produced `out.txt` and summarize it in 3 bullets.

Session: 7f3a…-session-id
```

This rendered string is also recorded via `addInvokedSkill(...)` so it survives compaction
([`processSlashCommand.tsx:1152`](../src/utils/processUserInput/processSlashCommand.tsx#L1152)).

---

## 3. Step 3 — Building the messages (`getMessagesForPromptSlashCommand`)

The rendered text is wrapped into a small message set
([`processSlashCommand.tsx:1184`](../src/utils/processUserInput/processSlashCommand.tsx#L1184)):

```jsonc
[
  // (1) METADATA breadcrumb — formatCommandLoadingMetadata (processSlashCommand.tsx:1047)
  { "role": "user", "content": "<command-message>pdf-extract</command-message>\n<command-name>/pdf-extract</command-name>\n<command-args>report.pdf</command-args>" },

  // (2) MAIN CONTENT — the rendered SKILL.md body from Step 2, isMeta:true
  { "role": "user", "content": [ { "type": "text", "text": "Base directory for this skill: …(full rendered body)…" } ] },

  // (3) ATTACHMENT MESSAGES — from @-mentions / MCP resources referenced in the body
  //     (skipSkillDiscovery:true so the SKILL.md text itself doesn't trigger discovery)

  // (4) command_permissions attachment — carries allowedTools + model (state only)
  { "type": "attachment", "attachment": { "type": "command_permissions", "allowedTools": ["Bash","Read"], "model": undefined } }
]
```

- **(1) metadata** uses `<command-name>/pdf-extract</command-name>` for user-invocable skills, or
  `<command-name>name</command-name>` + `<skill-format>true</skill-format>` for model-only skills
  (`formatSkillLoadingMetadata`, [`processSlashCommand.tsx:1020`](../src/utils/processUserInput/processSlashCommand.tsx#L1020)).
  This is the `<command-name>` tag the Skill prompt refers to — its presence tells the model the
  skill is already loaded, so it should **follow the body, not call `Skill` again**.
- **(4) `command_permissions`** renders to **no API message** — `getAttachmentMessageText` returns
  `[]` for it ([`messages.ts:4672`](../src/utils/messages.ts#L4672)). Its `allowedTools`/`model` are
  applied to the run via `contextModifier` instead (Step 4).

---

## 4. Step 4 — The tool result + injected messages

`SkillTool.call` returns three things ([`SkillTool.ts:770`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L770)):

1. **The `tool_result`** for the `Skill` call itself — a short string
   ([`SkillTool.ts:861`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L861)):

   ```text
   Launching skill: pdf-extract
   ```

   (forked skills instead return `Skill "<name>" completed (forked execution).\n\nResult:\n<…>`).

2. **`newMessages`** — the messages from Step 3, **with the `<command-message>` metadata filtered
   out** (the tool handles that display) and each tagged with the `sourceToolUseID` so they stay
   transient until the tool resolves ([`SkillTool.ts:739`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L739)).
   In practice the injected message is the **rendered SKILL.md body (message 2)**.

3. **`contextModifier`** — applies the skill's `allowedTools` as permission allow-rules and any
   `model` / `effort` override for subsequent turns
   ([`SkillTool.ts:779`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L779)).

---

## 5. Step 5 — The next LLM call (the "whole prompt" after invocation)

The system prompt is unchanged (cached — see [`claude-code-system-prompt.md`](./claude-code-system-prompt.md)).
The `messages` array now ends like this — **this is the full prompt the model receives to act on
the skill**:

```jsonc
[
  /* … prior conversation … */

  // The assistant turn that called the skill
  {
    "role": "assistant",
    "content": [
      { "type": "text", "text": "I'll extract the PDF text." },
      { "type": "tool_use", "id": "toolu_SKILL1", "name": "Skill",
        "input": { "skill": "pdf-extract", "args": "report.pdf" } }
    ]
  },

  // The Skill tool_result (short ack)
  {
    "role": "user",
    "content": [ { "type": "tool_result", "tool_use_id": "toolu_SKILL1", "content": "Launching skill: pdf-extract" } ]
  },

  // The injected SKILL.md body (isMeta) — THIS is what drives the model's next actions
  {
    "role": "user",
    "content": [ { "type": "text", "text":
      "Base directory for this skill: /Users/you/.claude/skills/pdf-extract\n\n# PDF Extract\n\nYou are extracting text from the PDF at `report.pdf`.\n\nCurrent time for the log header: 14:07\n\nSteps:\n1. Run the bundled extractor script:\n\n   ```\n   python /Users/you/.claude/skills/pdf-extract/scripts/extract.py \"report.pdf\"\n   ```\n\n2. Read the produced `out.txt` and summarize it in 3 bullets.\n\nSession: 7f3a…-session-id"
    } ]
  }
]
```

The model reads the body and proceeds — i.e. it now issues a `Bash` call to run the script in
step 1, then a `Read` call for `out.txt`. Those tool calls are permitted because the skill's
`allowed-tools: Bash, Read` were applied via `contextModifier`.

---

## 6. How Scripts Inside a Skill Get Executed — the two mechanisms

This is the crux of "how the LLM executes the different scripts inside a skill." There are **two
separate execution paths**, and they happen at different times:

### Mechanism A — Load-time inline shell (`` !`…` `` / ` ```! … ``` `)

- **Who runs it:** Claude Code itself, during Step 2d, *before the model sees the text*.
- **How:** `executeShellCommandsInPrompt` matches the patterns, permission-checks each command,
  runs it through `BashTool`/`PowerShellTool`, and **substitutes the stdout inline** into the body.
- **Model's view:** the command is already gone — it only sees the *output* (`14:07` in our case).
- **Shell choice:** `shell:` frontmatter (`bash` default / `powershell` if the runtime gate allows).
- **Security:** MCP-sourced skills are never shell-executed; each command still goes through the
  normal permission system.
- **Use it for:** stamping dynamic values into the prompt (dates, `git status`, file lists).

### Mechanism B — Model-time scripts (the body tells the model to run them)

- **Who runs it:** the **model**, on its own turn, by issuing `Bash`/`Read`/etc. tool calls after
  reading the rendered body.
- **How the path resolves:** `${CLAUDE_SKILL_DIR}` was replaced (Step 2c) with the skill's
  absolute directory, and `Base directory for this skill: …` is prepended (Step 2a), so script
  references like `python ${CLAUDE_SKILL_DIR}/scripts/extract.py` become real, runnable absolute
  paths the model can pass straight to `Bash`.
- **Permissions:** the skill's `allowed-tools` are applied as allow-rules via `contextModifier`,
  so the model's `Bash`/`Read` calls for the skill's scripts don't re-prompt.
- **Bundled skills:** packaged reference files/scripts are extracted to disk on first invocation so
  those paths exist (see `BundledSkillDefinition.files`,
  [`src/skills/bundledSkills.ts`](../src/skills/bundledSkills.ts)).
- **Use it for:** the actual work — running extractor scripts, test runners, generators, etc.

| | Mechanism A (`!` blocks) | Mechanism B (script instructions) |
|--|--------------------------|-----------------------------------|
| Executed by | Claude Code (load time) | The model (its own tool calls) |
| When | Before the body reaches the model | After the model reads the body |
| Appears in prompt as | The command's **output** | Literal instructions + resolved paths |
| Enabling glue | `executeShellCommandsInPrompt` | `${CLAUDE_SKILL_DIR}` + base-dir line + `allowed-tools` |
| MCP skills | Disabled | Allowed (no path injection) |

---

## 7. Inline vs. Forked Skills

- **Inline** (`context: inline`, default) — everything above: the body is injected into the
  **current** conversation and the main model executes it.
- **Forked** (`context: fork` + `agent: <type>`) — `executeForkedSkill`
  ([`SkillTool.ts:122`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L122)) runs the
  rendered body as the prompt of an **isolated sub-agent** with its own context/budget; only the
  final result returns to the main thread (as the `tool_result` text in Step 4, variant 1).

---

## 8. Step-by-Step Summary

| Step | What happens | Code |
|------|--------------|------|
| 1 | Model calls `Skill({skill,args})`; validate + permission | [`SkillTool.ts:358`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L358) / [`:436`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L436) |
| 2a | Prepend `Base directory for this skill:` | [`loadSkillsDir.ts:345`](../src/skills/loadSkillsDir.ts#L345) |
| 2b | Substitute `$file` / `$ARGUMENTS` / `$N` | [`argumentSubstitution.ts:94`](../src/utils/argumentSubstitution.ts#L94) |
| 2c | Replace `${CLAUDE_SKILL_DIR}` / `${CLAUDE_SESSION_ID}` | [`loadSkillsDir.ts:359`](../src/skills/loadSkillsDir.ts#L359) |
| 2d | Run `` !`…` `` blocks now, inline their output | [`promptShellExecution.ts:69`](../src/utils/promptShellExecution.ts#L69) |
| 3 | Build metadata + body + permissions messages | [`processSlashCommand.tsx:1184`](../src/utils/processUserInput/processSlashCommand.tsx#L1184) |
| 4 | Return `tool_result` "Launching skill: X" + newMessages + contextModifier | [`SkillTool.ts:770`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L770) / [`:861`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L861) |
| 5 | Next LLM call: body in context; model runs scripts via Bash/Read | [`src/query.ts:884`](../src/query.ts#L884) |

The "whole prompt" added by a skill invocation is therefore: a **`Launching skill: <name>`** tool
result, followed by the **fully-rendered `SKILL.md` body** (base-dir line + substituted args +
resolved `${…}` vars + inlined `!`-command output`) injected as a user message — after which the
model runs the skill's scripts itself with the skill's granted tools.
