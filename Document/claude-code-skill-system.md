# Claude Code — The Skill System

> A code-level map of how Claude Code's **skills** work: what a skill is, where skills come
> from, how they're loaded, how they're **found** (auto-discovery + search), and how they're
> **invoked** (the `Skill` tool, inline vs. forked execution).
>
> Companion diagrams: [`claude-code-skill-system.drawio`](./claude-code-skill-system.drawio)
> (4 pages: *Loading Pipeline*, *Discovery & Search*, *Invocation Flow*, *Data Model & Sources*).

All links are relative to the repository root. `src/foo.ts:42` = file `src/foo.ts`, line `42`.

---

## 1. What a Skill Is

A **skill** is a Markdown file (`SKILL.md`) with YAML frontmatter plus an optional bundle of
resources (scripts, reference docs). At runtime each skill is normalized into a
`PromptCommand` ([`src/types/command.ts:25`](../src/types/command.ts#L25)) — a member of the
`Command` union — so skills, slash commands, and MCP "skills" all flow through one registry.

A skill can be surfaced three ways:
1. **Typed by the user** as a slash command (`/commit`) — if `user-invocable`.
2. **Invoked by the model** via the `Skill` tool — if not `disable-model-invocation`.
3. **Auto-surfaced / auto-loaded** by skill search based on the user's intent.

```
SKILL.md (frontmatter + body)
        │  parseFrontmatter + parseSkillFrontmatterFields
        ▼
   PromptCommand  ──►  command registry (getCommands)
        │                     │
        ├── slash picker ◄─────┤ (user-invocable)
        ├── Skill tool   ◄─────┤ (model-invocable)
        └── skill search ◄─────┘ (TF-IDF index)
```

---

## 2. The Skill Data Model

### 2.1 Frontmatter fields

Parsed by `parseSkillFrontmatterFields()`
([`src/skills/loadSkillsDir.ts:185`](../src/skills/loadSkillsDir.ts#L185)); raw YAML parsing in
[`src/utils/frontmatterParser.ts`](../src/utils/frontmatterParser.ts).

| Frontmatter key | Meaning |
|-----------------|---------|
| `name` | Display name override |
| `description` | Short summary (falls back to first body line) |
| `when_to_use` | Detailed trigger guidance (heavily weighted in search) |
| `allowed-tools` | Tools the skill may use (becomes permission allow-rules) |
| `argument-hint` / `arguments` | Arg UI hint and named-arg list for substitution |
| `user-invocable` | Whether a user can type `/name` (default true) |
| `disable-model-invocation` | If true, the model cannot call it via `Skill` |
| `model` | Model override (`haiku`/`sonnet`/`opus`/id/`inherit`) |
| `effort` | Effort level for forked agents |
| `context` | `inline` (expand into chat) or `fork` (run in sub-agent) |
| `agent` | Agent type used when `context: fork` |
| `paths` | Glob(s) — conditional activation when matching files are touched |
| `shell` | Shell for inline `` !`…` `` blocks (`bash`/`powershell`) |
| `hooks` | Hooks registered while the skill is active |
| `version` | Semantic version |

### 2.2 The `PromptCommand` type

Defined at [`src/types/command.ts:25`](../src/types/command.ts#L25). Key runtime fields beyond
the frontmatter: `source` / `loadedFrom` (provenance), `skillRoot` (resource base dir),
`contentLength` (token estimation), `isHidden` (slash-picker visibility = `!userInvocable`),
and the loader callback `getPromptForCommand(args, context)` which renders the body into
`ContentBlockParam[]` at invocation time.

`LoadedFrom` provenance enum: [`src/skills/loadSkillsDir.ts:67`](../src/skills/loadSkillsDir.ts#L67)
— `bundled | skills | commands_DEPRECATED | plugin | managed | mcp`.

---

## 3. Where Skills Come From (Sources)

Skills are aggregated by `getCommands()`
([`src/commands.ts:553`](../src/commands.ts#L553)). Sources, in precedence order:

| # | Source | Location / loader | Reference |
|---|--------|-------------------|-----------|
| 1 | **Bundled** | compiled into the binary, `registerBundledSkill()` | [`src/skills/bundledSkills.ts:43`](../src/skills/bundledSkills.ts#L43) |
| 2 | **Managed / policy** | `{managed}/.claude/skills/` (gated by `CLAUDE_CODE_DISABLE_POLICY_SKILLS`) | [`src/skills/loadSkillsDir.ts:640`](../src/skills/loadSkillsDir.ts#L640) |
| 3 | **User** | `~/.claude/skills/` | [`src/skills/loadSkillsDir.ts:689`](../src/skills/loadSkillsDir.ts#L689) |
| 4 | **Project** | `.claude/skills/` (walked up to home) | [`src/skills/loadSkillsDir.ts:692`](../src/skills/loadSkillsDir.ts#L692) |
| 5 | **Additional dirs** | `--add-dir`/`.claude/skills/` | [`src/skills/loadSkillsDir.ts:692`](../src/skills/loadSkillsDir.ts#L692) |
| 6 | **Legacy commands** | `.../commands/*.md` (dir or single file) | [`src/skills/loadSkillsDir.ts:566`](../src/skills/loadSkillsDir.ts#L566) |
| 7 | **Plugin skills** | `plugin.skillsPath` / `skillsPaths[]`, namespaced `plugin:skill` | [`src/utils/plugins/loadPluginCommands.ts:840`](../src/utils/plugins/loadPluginCommands.ts#L840) |
| 8 | **Built-in plugin skills** | `BuiltinPluginDefinition.skills` | [`src/plugins/builtinPlugins.ts:108`](../src/plugins/builtinPlugins.ts#L108) |
| 9 | **MCP skills** | resources with `skill://` URI, namespaced `mcp__server__skill` | [`src/skills/mcpSkills.ts:30`](../src/skills/mcpSkills.ts#L30) |
| 10 | **Dynamic / conditional** | discovered at runtime when files are touched | [`src/skills/loadSkillsDir.ts:861`](../src/skills/loadSkillsDir.ts#L861) |

Deduplication is by resolved real path (handles symlinks/overlapping dirs); first-wins by the
precedence order above. See **Page 4 – "Data Model & Sources"**.

---

## 4. The Loading Pipeline

The on-disk loader entry point is `getSkillDirCommands(cwd)`
([`src/skills/loadSkillsDir.ts:638`](../src/skills/loadSkillsDir.ts#L638)), aggregated together
with plugin / bundled / MCP skills by `getCommands()`. See **Page 1 – "Loading Pipeline"**.

```
getCommands(cwd)                              src/commands.ts:553
  └─ loadAllCommands(cwd)  [memoized]
       └─ getSkills(cwd)   [parallel loaders]
            ├─ getSkillDirCommands(cwd)        loadSkillsDir.ts:638
            │     ├─ loadSkillsFromSkillsDir()  (managed / user / project / add-dir)
            │     ├─ loadSkillsFromCommandsDir() (legacy /commands)
            │     ├─ parseFrontmatter → parseSkillFrontmatterFields  (:185)
            │     ├─ createSkillCommand()        (:270)
            │     ├─ dedupe by realpath
            │     └─ split: unconditional vs conditional (paths)
            ├─ getPluginSkills()  [memoized]    loadPluginCommands.ts:840
            ├─ getBundledSkills()               bundledSkills.ts
            └─ getBuiltinPluginSkillCommands()  builtinPlugins.ts
       + getDynamicSkills()  (runtime-activated)
       + filter by isEnabled() & availability, dedupe
```

Each parsed skill becomes a `PromptCommand` via `createSkillCommand()`
([`src/skills/loadSkillsDir.ts:270`](../src/skills/loadSkillsDir.ts#L270)).

### 4.1 Conditional (path-gated) skills

Skills with a `paths:` glob are loaded but held **pending** in a `conditionalSkills` map. When
a file operation touches a matching path, two functions activate them:
- `discoverSkillDirsForPaths()` ([`src/skills/loadSkillsDir.ts:861`](../src/skills/loadSkillsDir.ts#L861)) — walks up from touched files to find new `.claude/skills/` dirs.
- `activateConditionalSkillsForPaths()` ([`src/skills/loadSkillsDir.ts:997`](../src/skills/loadSkillsDir.ts#L997)) — gitignore-style matches `paths` and promotes pending → `dynamicSkills`, emitting a `skillsLoaded` signal that busts the command cache.

### 4.2 Turning skills into the model's surfaces

| Function | Location | Produces |
|----------|----------|----------|
| `getSkillToolCommands(cwd)` | [`src/commands.ts:640`](../src/commands.ts#L640) | model-invocable skills listed in the `Skill` tool prompt |
| `getSlashCommandToolSkills(cwd)` | [`src/commands.ts:663`](../src/commands.ts#L663) | described skills for the slash-command picker |
| `findCommand(name, commands)` | [`src/commands.ts:774`](../src/commands.ts#L774) | resolves a name/alias/userFacingName to a command |

---

## 5. How a Skill Is Found (Discovery & Search)

There are two finding mechanisms: **explicit search** (the model or user searches) and
**proactive discovery** (the system surfaces skills based on intent). Both share one TF-IDF
index. See **Page 2 – "Discovery & Search"**.

### 5.1 The TF-IDF index

Built and cached per-cwd by `getSkillIndex()`
([`src/services/skillSearch/localSearch.ts:298`](../src/services/skillSearch/localSearch.ts#L298)).
It is a **weighted TF-IDF** index (not embeddings):

- Tokenize + stem each field — `tokenize()` ([`localSearch.ts:151`](../src/services/skillSearch/localSearch.ts#L151)), `tokenizeAndStem()` ([`:201`](../src/services/skillSearch/localSearch.ts#L201)); supports CJK bi-grams.
- Field weights: `name 3.0`, `whenToUse 2.0`, `description 1.0`, `allowedTools 0.3` — `computeWeightedTf()` ([`:212`](../src/services/skillSearch/localSearch.ts#L212)).
- Score by cosine similarity of query vs. doc TF-IDF vectors — `cosineSimilarity()` ([`:249`](../src/services/skillSearch/localSearch.ts#L249)).

Query execution: `searchSkills(query, index)`
([`localSearch.ts:383`](../src/services/skillSearch/localSearch.ts#L383)) — builds the query
TF-IDF vector, scores all docs, applies a name-match bonus and a display min-score threshold,
returns the top-N.

### 5.2 Intent normalization (cross-language)

Because Chinese queries vs. English skill text break TF-IDF (CJK terms look artificially rare),
`normalizeQueryIntent()`
([`src/services/skillSearch/intentNormalize.ts`](../src/services/skillSearch/intentNormalize.ts))
asks **Claude Haiku** to reduce the query to 3–6 English keywords before search. ASCII-only
queries skip Haiku; results are LRU-cached; gated by `SKILL_SEARCH_INTENT_ENABLED`.

### 5.3 Proactive discovery & auto-loading

Discovery runs at two moments and emits a `skill_discovery` attachment:

| Trigger | Function | Location | Timing |
|---------|----------|----------|--------|
| Turn-zero (user input) | `getTurnZeroSkillDiscovery()` | [`prefetch.ts:308`](../src/services/skillSearch/prefetch.ts#L308) | blocking, called from attachments |
| Inter-turn (assistant turn) | `startSkillDiscoveryPrefetch()` | [`prefetch.ts:249`](../src/services/skillSearch/prefetch.ts#L249) | non-blocking, spawned from `query.ts:483` |

Helpers: `extractQueryFromMessages()` ([`prefetch.ts:54`](../src/services/skillSearch/prefetch.ts#L54)),
and `enrichResultsForAutoLoad()` ([`prefetch.ts:127`](../src/services/skillSearch/prefetch.ts#L127)),
which pulls the **full `SKILL.md` body** for top-scoring matches and marks them `autoLoaded`.

**Auto-load vs. recommend** — surfaced into a `<system-reminder>` (rendering in
[`src/utils/messages.ts:3876`](../src/utils/messages.ts#L3876)):
- score ≥ `SKILL_SEARCH_AUTOLOAD_MIN_SCORE` (default 0.30, up to 2 skills): body injected inline in `<loaded-skill>` tags — *"apply now, don't call `Skill` again."*
- below that: listed as recommendations the model may invoke via `Skill`.

Session dedupe (`discoveredThisSession` set) prevents re-surfacing the same skill twice.

### 5.4 Explicit discovery tool

`DiscoverSkillsTool`
([`packages/builtin-tools/src/tools/DiscoverSkillsTool/DiscoverSkillsTool.ts`](../packages/builtin-tools/src/tools/DiscoverSkillsTool/DiscoverSkillsTool.ts))
lets the model search on demand — it calls the same `getSkillIndex()` + `searchSkills()`.

### 5.5 Feature flags & key env vars

| Flag / env | Effect | Reference |
|------------|--------|-----------|
| `EXPERIMENTAL_SKILL_SEARCH` | build-time: compile search in | [`src/services/skillSearch/featureCheck.ts`](../src/services/skillSearch/featureCheck.ts) |
| `SKILL_SEARCH_ENABLED=1` | runtime: activate prefetch/discovery | featureCheck.ts |
| `SKILL_SEARCH_INTENT_ENABLED=1` | enable Haiku intent normalization | intentNormalize.ts |
| `SKILL_SEARCH_AUTOLOAD_MIN_SCORE` / `_LIMIT` / `_MAX_CHARS` | auto-load thresholds (0.30 / 2 / 12000) | prefetch.ts |
| `SKILL_LEARNING_ENABLED=1` | record skill gaps for learning | [`src/services/skillLearning/featureCheck.ts`](../src/services/skillLearning/featureCheck.ts) |

---

## 6. How a Skill Is Invoked

The `Skill` tool drives invocation. See **Page 3 – "Invocation Flow"**.

### 6.1 The `Skill` tool

Defined in
[`packages/builtin-tools/src/tools/SkillTool/SkillTool.ts`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts):

- **Name** `Skill` ([`:336`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L336)), constant `SKILL_TOOL_NAME` in [`constants.ts`](../packages/builtin-tools/src/tools/SkillTool/constants.ts).
- **Input schema** ([`:295`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L295)): `{ skill: string, args?: string }`.
- **Prompt/description** `getPrompt()` ([`prompt.ts:174`](../packages/builtin-tools/src/tools/SkillTool/prompt.ts#L174)) — tells the model that slash commands *are* skills, that matching a skill is a **blocking requirement** to call this tool first, and to not re-invoke a skill already loaded (signalled by a `<command-name>` tag).

The skill **listing budget** in that prompt is ~1% of the context window
(`SKILL_BUDGET_CONTEXT_PERCENT`, [`prompt.ts:21`](../packages/builtin-tools/src/tools/SkillTool/prompt.ts#L21)),
per-skill descriptions truncated to 1536 chars.

### 6.2 Validation → permission → execution

| Step | Method | Location |
|------|--------|----------|
| Validate skill exists, enabled, prompt-type | `validateInput()` | [`SkillTool.ts:358`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L358) |
| Deny/allow rules, safe-property auto-allow, else ask | `checkPermissions()` | [`SkillTool.ts:436`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L436) |
| Locate skill | `getAllCommands()` + `findCommand()` | [`SkillTool.ts:81`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L81) |
| Route: fork vs. inline | `command.context === 'fork'` check | [`SkillTool.ts:122`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L122) |

### 6.3 Inline execution (default)

The skill body is rendered and injected into the **current** conversation:

1. `processPromptSlashCommand()` → `getMessagesForPromptSlashCommand()` ([`src/utils/processUserInput/processSlashCommand.tsx`](../src/utils/processUserInput/processSlashCommand.tsx)).
2. `command.getPromptForCommand(args, context)` ([`src/skills/loadSkillsDir.ts:344`](../src/skills/loadSkillsDir.ts#L344)) renders the body:
   - prepends the skill base directory,
   - substitutes `$ARGUMENTS` / named args (`substituteArguments`),
   - replaces `${CLAUDE_SKILL_DIR}` and `${CLAUDE_SESSION_ID}`,
   - executes inline `` !`…` `` shell blocks (except MCP skills).
3. The rendered text is wrapped as user messages marked `isMeta: true`, plus a
   `command_permissions` attachment carrying `allowedTools`/`model`.
4. `addInvokedSkill()` records the skill so its content **survives compaction** and is restored
   post-compact.
5. The tool returns `newMessages` + a `contextModifier` that applies the skill's `allowedTools`
   allow-rules and any `model`/`effort` override ([`SkillTool.ts:771`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L771)).

### 6.4 Forked execution (`context: fork`)

`executeForkedSkill()`
([`SkillTool.ts:122`](../packages/builtin-tools/src/tools/SkillTool/SkillTool.ts#L122)) runs the
skill in an **isolated sub-agent** (agent type from the `agent` frontmatter) with its own token
budget; only the final result returns to the main thread.

### 6.5 Argument substitution

`getPromptForCommand` calls `substituteArguments()`
([`src/utils/argumentSubstitution.ts`](../src/utils/argumentSubstitution.ts)) to replace
`$ARGUMENTS` and named placeholders before the body enters context.

---

## 7. End-to-End: From User Message to Applied Skill

```
User message
   │
   ├─(A) turn-zero skill discovery  (getTurnZeroSkillDiscovery, prefetch.ts:308)
   │       └─ normalize intent → searchSkills → enrichForAutoLoad
   │             └─ skill_discovery attachment
   │                   ├─ score≥0.30 → <loaded-skill> body injected (apply now)
   │                   └─ else       → recommended list (model may call Skill)
   │
   ├─(B) model decides a skill matches  →  Skill tool call { skill, args }
   │       ├─ validateInput  →  checkPermissions
   │       ├─ inline:  getPromptForCommand → substitute → inject isMeta msgs
   │       │            + contextModifier (allowedTools / model / effort)
   │       │            + addInvokedSkill (survives compaction)
   │       └─ fork:    executeForkedSkill → sub-agent → final result
   │
   └─(C) inter-turn prefetch (startSkillDiscoveryPrefetch, prefetch.ts:249)
           └─ background discovery on write-pivot turns, deduped per session
```

---

## 8. Quick Reference — Key Files

| Concern | File |
|---------|------|
| Skill types | [`src/types/command.ts`](../src/types/command.ts) |
| Disk loading + conditional skills | [`src/skills/loadSkillsDir.ts`](../src/skills/loadSkillsDir.ts) |
| Bundled skills | [`src/skills/bundledSkills.ts`](../src/skills/bundledSkills.ts) |
| MCP skills | [`src/skills/mcpSkills.ts`](../src/skills/mcpSkills.ts) |
| Plugin skills | [`src/utils/plugins/loadPluginCommands.ts`](../src/utils/plugins/loadPluginCommands.ts) |
| Command aggregation / filtering | [`src/commands.ts`](../src/commands.ts) |
| Frontmatter parsing | [`src/utils/frontmatterParser.ts`](../src/utils/frontmatterParser.ts) |
| TF-IDF search index | [`src/services/skillSearch/localSearch.ts`](../src/services/skillSearch/localSearch.ts) |
| Intent normalization (Haiku) | [`src/services/skillSearch/intentNormalize.ts`](../src/services/skillSearch/intentNormalize.ts) |
| Proactive discovery / auto-load | [`src/services/skillSearch/prefetch.ts`](../src/services/skillSearch/prefetch.ts) |
| Discover tool | [`packages/builtin-tools/src/tools/DiscoverSkillsTool/`](../packages/builtin-tools/src/tools/DiscoverSkillsTool) |
| Skill tool (invocation) | [`packages/builtin-tools/src/tools/SkillTool/`](../packages/builtin-tools/src/tools/SkillTool) |
| Inline render / message build | [`src/utils/processUserInput/processSlashCommand.tsx`](../src/utils/processUserInput/processSlashCommand.tsx) |
| Discovery → system-reminder | [`src/utils/messages.ts`](../src/utils/messages.ts) |

See also the agent-loop companion doc: [`claude-code-agent-loop.md`](./claude-code-agent-loop.md).
