[Back to Main README](../README.md) · [中文版](build-your-own-agent_zh.md)

# Build Your Own AI Agent: A Design Space Guide

> A guide to the design decisions behind a production AI agent, starting from our analysis of Claude Code and extending it through other systems and recent research.

Every production coding agent must answer the same recurring design questions. Claude Code is one set of answers. This guide maps the choices, explains where they can be combined, and asks what evidence would justify them in your own system.

**Scope:** Claude Code architecture examples refer to the v2.1.88 snapshot analyzed in this repository. Later product releases and cross-system comparisons are identified separately; sources were reviewed through September 15, 2026. See the [architecture analysis](architecture.md) and [source notes](agent-design-space-source-notes.md) for evidence and limitations.

---

## Decision 1: Where Does Reasoning Live?

**The question:** How much decision-making do you put in the model vs. in your harness code?

| Approach | Example | Trade-off |
|:---------|:--------|:----------|
| **Model-led loop** | Claude Code v2.1.88 (~1.6% AI decision logic in our code classification) | Gives the model latitude while the harness manages execution, context, and permissions. Reliability still depends on the model and its environment. |
| **Explicit state graphs** | LangGraph | Makes state, dependencies, and transitions inspectable. Nodes and routing can still use model judgment; the graph needs maintenance as the workflow changes. |
| **Explicit planning and task tracking** | Multi-stage workflows | Makes intermediate goals visible. Plans must be revised when observations invalidate their assumptions. |

**Lesson from Claude Code:** A small reasoning loop can rely on substantial infrastructure. Context delivery, tool execution, recovery, and verification deserve their own design effort. Evaluate the model and harness together on your workload; neither code size nor a leaderboard gap determines the right amount of scaffolding.

**Where Graph Engineering fits:** The 2026 [Graph Engineering survey](https://arxiv.org/html/2608.21156v2#S4) organizes the problem around task structure, agent coordination, and runtime state. This framework helps builders make dependencies, responsibilities, and completion conditions explicit. It is broader than using a knowledge graph for retrieval, and it builds on earlier graph-based agent work.

A graph can define dependencies and acceptance conditions while an agent inside each node chooses its next action. For example, [LangGraph's agent example](https://docs.langchain.com/oss/python/langgraph/workflows-agents#agents) uses conditional edges to follow the model's decision to call a tool or stop. A practical design can combine that structure with a feedback loop and a harness that executes actions and records results. The choice is which relationships to make explicit, and which decisions to leave open.

**Keep earlier requirements in scope.** [LoopsBench](https://arxiv.org/html/2608.00267v1#S2.SS6) provides all attached tests from the start, but activates scoring for a development unit only after its prerequisites pass. Tests for completed units remain active as regression checks during later work. The agent remains free to choose where to edit.

**Use a graph to review completed work, too.** [Trace2Flow](https://arxiv.org/html/2609.13136v1#S4) is a research prototype that turns a completed agent trace into an editable workflow graph. Users can inspect step inputs and outputs, change dependencies, and rerun steps. A control graph guides execution; this graph organizes recorded work afterward. Keep the connection between each step and its trace evidence.

**Questions to ask yourself:**

- Which decisions benefit from model judgment, and how will you check the result?
- Which dependencies or approval gates must hold regardless of the model's plan?
- Do you need dynamic routing within a known graph, or does the graph itself need to change?

---

## Decision 2: What Is Your Safety Posture?

**The question:** How do you prevent the agent from doing harmful things?

| Approach | Example | Trade-off |
|:---------|:--------|:----------|
| **Deny-first with layered enforcement** | Claude Code's permission architecture | Combines checks at several boundaries. Shared assumptions and failure modes still need examination. |
| **Container isolation** | SWE-Agent, OpenHands | Bounds access according to mounts, network settings, credentials, and privileges. The container configuration determines the actual boundary. |
| **VCS rollback** | Aider's Git integration | Helps recover tracked file changes. Does not undo network requests or other external side effects. |
| **Approval gates** | Interactive agent tools | Lets a user authorize an action. The preview must match what will execute, and repeated prompts consume attention. |

**Lesson from Claude Code:** Layered checks need explicit fallback behavior. If parsing, classification, or a budget limit prevents one check from finishing, decide whether the action is denied, isolated, or sent for review. The presence of several checks alone does not establish that they fail independently.

**Authorization has a scope and a lifetime.** Distinguish durable policy, operating mode, and a grant for a particular action or session. A delegated task or a resumed conversation should carry only the authority that remains valid for its current resources and environment. See [Decision 6](#decision-6-how-do-sessions-persist) for the persistence implications.

**Separate information access from action authority.** [Twin Agent](https://arxiv.org/html/2607.19595v1#S3.SS3) limits the length of hints that a reader sends from untrusted sources to an executor. Only the executor can perform privileged actions, and it cannot read those sources directly. Define each agent's inputs and permissions, then check what can pass between them.

[APPA v2](https://arxiv.org/html/2607.24625v2#S4) checks a tool call before execution and checks its actual return before the data enters context. It can use a temporary branch to inspect untrusted data, with a predefined return format. Rejecting returned data does not undo an external effect that already occurred.

**Questions to ask yourself:**

- What's the worst thing your agent could do: delete production data, send an unintended message, or disclose code?
- Can sandboxing reduce the number of decisions users must make?
- Does authorization cover the actual action, including its arguments and resource identity?
- What happens when a check times out or a child task requests broader access?

---

## Decision 3: How Do You Manage Context?

**The question:** The context window is finite. How do you decide what the model sees?

These mechanisms can be combined; they serve different purposes and lifetimes.

| Approach | Example | Trade-off |
|:---------|:--------|:----------|
| **Graduated compaction pipeline** | Claude Code's five-layer analysis | Tries smaller reductions before full summarization. More stages mean more interactions to test. |
| **Simple truncation** | Basic conversation histories | Easy to implement, but may discard important early context. |
| **Sliding window** | Chat applications | Predictable size, but does not identify semantic importance. |
| **Retrieval** | Repository search and RAG | Keeps detailed material outside the prompt. Results need relevance, provenance, and enough surrounding context. |
| **Summarization** | Long conversations | Reduces history size, but may lose constraints or unresolved work. |
| **Notes and history across context windows** | Codex experimental context management | Keeps a checkpoint and retrieves earlier details after a window reset. Recovery depends on the notes, record identifiers, and history service. |
| **Cross-session memory** | Local Codex memory | Reuses selected experience across tasks. Requires rules for writing, retrieval, correction, and forgetting. |

**Lesson from Claude Code:** Context scarcity shapes lazy loading, deferred tool schemas, subagent return formats, and tool-result budgets. Design the information flow before the window fills up.

**The graduated approach:** The analyzed Claude Code pipeline combines budget reduction, history trimming, cache-aware compression, virtual projection, and full summarization. Which stages run depends on their triggers and configuration; this is a version-specific implementation, not a mandatory sequence for every agent.

**Separate working-context management from cross-session experience reuse.** Codex's [local memories](https://learn.chatgpt.com/docs/customization/memories) extract experience from eligible prior chats. Within a running task, [summarizing compaction](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact.rs#L63) and an experimental notes-and-history mode offer different ways to cross a context boundary. The [CLI command name `/compact`](https://learn.chatgpt.com/docs/developer-commands?surface=cli) alone does not tell you which path runs.

The experimental switch, [introduced in CLI 0.153.0](https://github.com/openai/codex/releases/tag/rust-v0.153.0), enables token-budget guidance, history notes, and `new_context` when turned on for eligible ChatGPT Plus, Pro, or Pro Lite sessions on the Codex backend. The protocol asks the agent to save a checkpoint, start a fresh window, and recover details through notes and history. In v0.153.4, [this compaction branch](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact_token_budget.rs) skips model/server summarization but preserves compact hooks and lifecycle events. Later main-branch code adds a model-capability check and marks Astra as supported; [the versioned account](./agent-design-space-source-notes.md#source-codex-context-management) separates that change from the released snapshot.

This design separates editable working notes from retrievable history. It makes checkpoint quality, source references, and recovery behavior part of the agent's control policy. Evaluate whether goals, constraints, permissions, and evidence survive a window reset and remain valid; having a history store does not guarantee that the agent retrieves the right details.

For reusable experience, define what qualifies for storage, when it should be rechecked, and how corrections reach future retrieval. Codex's [consolidation instructions](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/memories/write/templates/memories/consolidation.md#L157) ask the model to remove guidance supported only by deleted inputs while preserving guidance with surviving support. That is an update protocol to test, not a guarantee that every stale memory will disappear. Keep mandatory rules in explicit policy or instruction files.

Codex's September 8 [main-branch update](https://github.com/openai/codex/commit/2cbbf0c9b542a36a1c3284b5e804917635b6f666) adds memory v2; the read path uses the selected version. The [stable 0.154.0 memory configuration](https://github.com/openai/codex/blob/6b9826e3aa83b1a5947db50f4332cb9c65f1b340/codex-rs/config/src/types.rs#L289) has no version selector.

**Define where memory can be used and who can change it.** The [v2 extraction instructions](https://github.com/openai/codex/blob/2cbbf0c9b542a36a1c3284b5e804917635b6f666/codex-rs/memories/write/templates/memories/stage_one_system_v2.md) ask the model to distinguish task-specific requests from lasting preferences and apply later corrections within the task. Claude Tag's [public-channel notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.268) can be recalled only within their channel, while workspace notes remain shared. In [Hermes Agent v0.21.2](https://github.com/NousResearch/hermes-agent/blob/939e45c91d751fadd94dcd1b873ac3cb44846213/tools/memory_tool.py#L129), the built-in memory tool requires approval to replace or remove memory during unattended background reviews.

**Questions to ask yourself:**

- What must survive compaction or a window reset: the goal, constraints, pending work, permissions, and evidence?
- Which information belongs in working state, execution logs, reusable memory, or mandatory instructions?
- How will a changed repository or a withdrawn source invalidate old guidance?

---

## Decision 4: How Do You Handle Extensibility?

**The question:** How do external tools, custom instructions, and user customizations plug into your system?

| Approach | Example | Trade-off |
|:---------|:--------|:----------|
| **Different extension surfaces** | Claude Code hooks, skills, plugins, and MCP | Supports lifecycle code, on-demand instructions, and external tools. Context cost depends on what is loaded, executed, and returned. |
| **Single unified API** | Tool-use frameworks | Gives extensions a common interface. Discovery, schema loading, and result size still need budgeting. |
| **Plugin marketplace** | IDE extensions | Makes distribution easier. Introduces decisions about source trust, updates, and execution privileges. |

**Lesson from Claude Code:** Load an extension's instructions and schemas when they become relevant. A command hook can handle an event without a model call, but content it returns to the conversation still uses context. Likewise, MCP cost depends on discovery and usage, rather than a fixed cost attached to the protocol.

**The three injection points:** A useful way to inspect an agent loop is to ask where an extension intervenes:

1. **assemble()** — What the model sees: instructions, tool schemas, and retrieved context.
2. **model()** — What actions the model can request through the exposed tools.
3. **execute()** — Whether and how an action runs: permission gates and pre/post hooks.

Treat these as conceptual boundaries; implementations need not use these function names. For each extension, distinguish permission to install or update it from permission to perform its actions.

**Define a tool operation across requests.** [MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog) removes protocol-level sessions; applications carry cross-call state through explicit handles. When a tool needs more input, the [multi round-trip protocol](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr) returns `input_required`; the client retries with `inputResponses` and returns any `requestState` unchanged. These are separate requests for one logical operation, so specify who owns its state and how retries handle effects that may have occurred.

**Bind command consent to the reviewed inputs.** In [Claude Code v2.1.271](https://code.claude.com/docs/en/plugins-reference#plugin-install), a plugin install or update can require approval for a marketplace command. When the command is shown but not run, the `--json` result includes it and its digest. The user reviews the command, then passes `--accept-command <sha256>` from their own terminal. Acceptance is bound to the command, plugin, and marketplace catalog; a change to any of them invalidates it.

**Questions to ask yourself:**

- How many tools will your agent expose, and when will their schemas enter context?
- Who can publish or update an extension, and how does that affect already-running sessions?
- Which process executes the extension, with which filesystem, network, and credential access?

---

## Decision 5: How Do Subagents Work?

**The question:** When the agent spawns sub-tasks, do they share context or run in isolation?

| Approach | Example | Trade-off |
|:---------|:--------|:----------|
| **Isolated context + selected results** | Claude Code's sidechain transcripts | Limits how much child history enters the parent context. Handoffs must preserve the evidence needed for the next decision. |
| **Shared context** | Multi-agent systems with shared histories | Makes information available to all participants, but adds context pressure and potential interference. |
| **Message passing** | Actor model systems | Gives communication explicit boundaries, but requires a protocol for progress, failure, and completion. |

**Lesson from Claude Code:** Context isolation can keep detailed exploration out of the parent conversation. It does not make delegation free: measure total model work, duplicated exploration, and coordination cost. Separate context windows also do not imply separate filesystems, processes, or permissions.

**Define the handoff as well as the role.** A task needs an owner, dependencies, an output, and conditions for accepting that output. In [Agent Graph v0.3.0](https://github.com/context4ai/agent-graph/blob/387f80db65bf20a61bc666b4fa885200fcedad08/src/evaluator.ts#L137), a nonterminal node with a `satisfiedBy` condition remains `unverified` if it is marked completed but the supplied facts do not satisfy that condition. This is one concrete implementation of a completion check; the host still has to supply trustworthy facts.

**Check concurrent changes before accepting them.** Separate worktrees let agents edit independently, but their changes can still conflict. The [Claim Plane prototype](https://arxiv.org/pdf/2607.21909v1) checks versioned change intents before writes and checks an expanded scope again before allowing the additional mutation. Task assignment, permission to write, and acceptance of the combined result need separate decisions.

**Choose who owns shared session state.** With [Agent Host Protocol](https://code.visualstudio.com/blogs/2026/08/26/agent-host-architecture), the host owns the session and sends snapshots and ordered updates to multiple clients. Each harness retains its own agent loop, context management, and tools. A common session interface therefore still needs clear rules for when client actions take effect.

**Specify when a message takes effect.** [VS Code 1.137](https://code.visualstudio.com/updates/v1_137#_agent-queued-messages) queues agent messages sent to a busy chat. It starts processing queued messages in send order after the active turn succeeds. Sending to another session through its [Agent Host tools](https://code.visualstudio.com/docs/agents/run/sessions/manage-sessions#_orchestrate-sessions-from-agent-host-sessions) requires user confirmation. Define delivery timing as well as permission to send.

**Decide whether a worker may question its brief.** In its [Fusion design account](https://cognition.com/blog/local-fusion), Cognition describes adjusting brief detail, how much the worker may question instructions, and exploration duties for each model pair. Include these choices in the delegation protocol and test them with the models and tasks you use.

**Questions to ask yourself:**

- Do child tasks need each other's history, or only specific artifacts and messages?
- Who may update shared state, and who verifies the result?
- What authority does a child inherit, and how is it limited to the delegated work?
- Can the parent steer or stop an active child, and distinguish partial output from completion?

---

## Decision 6: How Do Sessions Persist?

**The question:** What happens when a session ends? What carries over?

| Approach | Example | Trade-off |
|:---------|:--------|:----------|
| **Append-only JSONL** | Claude Code's session logs | Easy to inspect and useful for reconstructing recorded events. Does not capture every external side effect by itself. |
| **Database and checkpoints** | Persistent agent runtimes | Supports queries, coordination, and recovery points. Requires explicit schemas, retention rules, and recovery semantics. |
| **Stateless requests** | APIs used without persisted application state | Keeps each request simple. The application must add continuity, audit, and recovery when needed. |

**Restore task state with explicit authority.** Distinguish durable policy, operating mode, and temporary grants. The analyzed Claude Code snapshot marks its computer-use app allowlist and session bypass flag as nonpersistent; those examples do not establish a universal rule that every kind of permission state must be discarded. Later, [v2.1.246](https://github.com/anthropics/claude-code/releases/tag/v2.1.246) fixed plan mode being lost on resume in specific VS Code and headless entry points. Preserve intended restrictions and revalidate grants against their scope and lifetime.

**Persistence also needs a recovery protocol.** Record which task a worker owns, which effects have completed, and which results still need delivery. After a crash, define who may retry and how duplicate effects are prevented or reconciled. A checkpoint alone does not establish exactly-once execution.

For example, Temporal's Deep Agents integration, [announced as a pre-release](https://temporal.io/blog/durable-digest-august-2026), separates replayable Workflow state from model calls and external I/O in Activities. Its [integration guide](https://docs.temporal.io/develop/python/integrations/deepagents) requires real-I/O tools and backends to be wrapped appropriately. Durability depends on those execution boundaries.

**Distinguish a retry from new work.** [Resume Means Resume v3](https://arxiv.org/html/2608.03836v3#S3) separates ordinary resumption from an intended fork and checks whether an approval has already been consumed. Preserve operation identities when an external effect may have completed before its result was recorded. Also define what remains in progress during a pause: [Temporal's pre-release pause feature](https://docs.temporal.io/encyclopedia/workflow/workflow-pause) stops new dispatch while running Activities can still finish.

**Give schedules and active runs separate controls.** [VS Code Automations](https://code.visualstudio.com/docs/agents/run/automations) is in Preview. Disabling its schedule prevents future scheduled runs but leaves the active run in progress; stopping that session is a separate action. Scheduled work also requires an awake machine: Agent Host automations need a running host process, while other automations need a running VS Code window.

**Questions to ask yourself:**

- Can a new worker tell completed work from an action whose outcome is still unknown?
- If two workers resume the same task, which one has authority to continue?
- Does “finished” mean computation completed, a result was accepted, or the user received it?

---

<a id="the-meta-pattern-three-recurring-design-commitments"></a>

## Three Principles Across the Six Decisions

Three principles connect these decisions in Claude Code with the broader design space:

1. **Give each mechanism a clear boundary.** Specify what a context stage, permission check, or extension controls and how it fails.

2. **Keep decisions tied to inspectable evidence.** Logs, checkpoints, and memories serve different purposes. Preserve enough provenance to revisit a completion claim or correct learned guidance.

3. **Combine model judgment with explicit execution rules.** The model can choose actions within a node or loop while the system checks dependencies, enforces permissions, and evaluates whether results meet acceptance conditions.

---

## Validate the Whole Run

These decisions meet at the points where the system accepts work, resumes it, or learns from it. Define those checks before giving a loop more autonomy.

| Transition | Evidence to require |
|:-----------|:--------------------|
| **Advance or stop** | Current task requirements, results of the relevant checks, unresolved issues, and a reason to continue or stop. Reaching a budget limit is a stop reason, not proof of completion. |
| **Resume or retry** | Current task ownership, valid authority, and the status of prior side effects. Recheck evidence affected by changed inputs or environments. |
| **Retain a memory, skill, or harness change** | Supporting traces, an explicit scope, checks for regressions, and a way to revise or withdraw the update. |

Mastra Factory's current [board rules](https://factory.mastra.ai/configure/boards-and-rules) separate allowed stage transitions, approval policies, and entry and exit actions. Listing a transition does not move a card automatically. Test what triggers the move, whether approval is required, who provides it, and which actions run when the card leaves or enters a stage.

[LoopArena](https://arxiv.org/html/2608.28281v1#S2) evaluates control decisions with a fixed Worker; its read-only Reporter summarizes evidence and cannot run tests. [HarnessLens](https://arxiv.org/html/2608.27311v1#S4) selects checks around the behavior a proposed change should affect and keeps a held-out test set. For skill updates, distinguish the cases that decide acceptance from those reserved for final testing: [SkillAdam](https://arxiv.org/html/2609.08944v1#S5.SS3) reuses sampled cases for acceptance and keeps a separate test split.

**Specify what an update changes.** Memory, skills, runtime code, and model weights need different checks.

| Updated object | Representative mechanism | What to check |
|:---------------|:-------------------------|:--------------|
| **Retrievable memory** | [Living-Harness v2](https://arxiv.org/html/2607.26598v2) updates memory and a state graph after evaluated runs. Later tasks retrieve these records as guidance. | Evidence, scope, and later retrieval of corrected guidance. |
| **Skills and their relations** | [GSE](https://arxiv.org/html/2608.06153v1#S3) changes skill content and relations between skills, including dependencies and conflicts. | Replay cases for affected skills, then test on separate cases. |
| **Runtime code and tools** | [Better Harnesses, Smaller Models](https://arxiv.org/html/2607.08938v1#S3) changes tools, hooks, context handling, and subagents while keeping the task model fixed. | Task results and costs with the model that will use the changed harness. |
| **Model weights** | [Multi-Harness RL](https://arxiv.org/html/2609.04518v1#S3) trains on experience from several harnesses and tests on an unseen harness. | Gains on the training interfaces and transfer to a different interface. |

Record which cases produce an update, which decide whether to accept it, and which measure its final performance. Include failed candidates and evaluation runs in the update cost.

---

## Resources for Builders

- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) — Build a small coding agent step by step.
- [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) — A Rust reimplementation for studying alternative implementation choices.
- [Yuyz0112/claude-code-reverse](https://github.com/Yuyz0112/claude-code-reverse) — Inspect Claude Code's LLM interactions.
- [Haseeb Qureshi's architecture comparison](https://gist.github.com/Haseeb-Qureshi/2213cc0487ea71d62572a645d7582518) — Claude Code, Codex, Cline, and OpenCode compared architecturally.
- [OpenHands](https://github.com/All-Hands-AI/OpenHands) — An open-source coding agent platform with environment isolation.
- [SWE-Agent](https://github.com/SWE-agent/SWE-agent) — A coding agent implementation for studying the agent–computer interface.
- [Aider](https://github.com/Aider-AI/aider) — A coding assistant with Git-based change management.
