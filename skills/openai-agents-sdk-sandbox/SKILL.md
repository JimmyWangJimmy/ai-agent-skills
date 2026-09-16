---
name: openai-agents-sdk-sandbox
description: >-
  Use this when building or reviewing multi-agent workflows with the OpenAI
  Agents SDK — especially SandboxAgent workspace work, MCP tools, guardrails,
  handoffs, Realtime voice, or choosing vs LangGraph / Microsoft Agent Framework.
---

# OpenAI Agents SDK: Agents, Sandbox, Guardrails, Handoffs, MCP

## Source
- Type: github
- Title: OpenAI Agents SDK (Python) — openai/openai-agents-python
- Link: https://github.com/openai/openai-agents-python
- Docs: https://openai.github.io/openai-agents-python/
- Stars (approx., from GitHub page at skill authoring): ~29.5k
- One-line gist: A lightweight multi-agent runtime where an Agent is instructions + tools + guardrails + handoffs; SandboxAgent adds an isolated mutable workspace for long-horizon file/shell work; MCP and Realtime plug into the same mental model.

## When to use / when NOT to use

**Use when:**
- You want a fast path to a working agent loop (Runner turns, tools, sessions, tracing) without designing a full state graph first.
- Work is **workspace-centric**: inspect repos, edit files, run commands, resume from snapshots — reach for `SandboxAgent` + `Manifest` + sandbox client.
- You need built-in **handoffs** (specialist takes over the conversation) or **agents-as-tools** (manager keeps control).
- You wire **MCP** servers (stdio / Streamable HTTP / SSE / HostedMCPTool) alongside function tools.
- You need **input/output/tool guardrails** with tripwires, or server-side **RealtimeAgent** voice over WebSocket.

**Do NOT use when:**
- The product requires an **explicit durable state graph** with node-level checkpointing, time-travel, and crash-resume as the primary control model → prefer **LangGraph** (or wrap the SDK in durable infrastructure).
- You are standardized on **Azure / .NET / Semantic Kernel / AutoGen migration** and Microsoft governance tooling → evaluate **Microsoft Agent Framework** first.
- You only need a single LLM call with no tools, no workspace, and no multi-agent routing — Responses/Chat Completions alone is enough.
- Sandbox is in **beta**: treat API/defaults as moving; pin versions and re-read docs before production contracts.

## Problem (plain language)
Product agents are not “one prompt.” They must call tools, sometimes edit real files, sometimes hand work to specialists, and sometimes speak in realtime — while staying safe against bad inputs/outputs. Teams either under-structure (prompt spaghetti) or over-structure (heavy graphs before the first loop works).

The OpenAI Agents SDK’s bet: keep orchestration **agent-centric** (who is active, which tools, which checks) and add a hard **execution boundary** only when the agent must mutate a filesystem (sandbox session). MCP standardizes tool plugs; guardrails and HITL gate side effects; handoffs vs as_tool choose control topology.

## Core mechanism (how it works)

### 1) Agent = instructions + tools + optional runtime behavior
An `Agent` is an LLM configured with:
- `instructions` (or dynamic instructions / prompt templates)
- `tools` (function tools, hosted tools) and/or `mcp_servers`
- optional `handoffs`, `input_guardrails` / `output_guardrails`, `output_type`, hooks, `tool_use_behavior`

`Runner.run` / `run_sync` / `run_streamed` owns the turn loop, tool execution, sessions, and tracing.

### 2) Two multi-agent control patterns
| Pattern | Control | Turn accounting |
|---------|---------|-----------------|
| **Handoffs** | Peer specialist **takes over** the same run | One top-level turn loop; active agent changes |
| **Agents as tools** | Manager **retains** conversation; specialist is a nested run | Nested max_turns/approvals; outer turn counter does not absorb nested turns |

Use handoffs for triage → billing/refund style ownership transfer. Use as_tool when the orchestrator must synthesize multiple specialist results.

### 3) SandboxAgent = Agent + workspace boundary
`SandboxAgent` keeps the full Agent surface, then adds:
- `default_manifest` — fresh-session workspace contract (files, `GitRepo`, mounts, env)
- `capabilities` — sandbox-native behavior (`Filesystem`, `Shell`, `Skills`, `Memory`, `Compaction`; default includes Filesystem + Shell + Compaction)
- `run_as` — identity for model-facing sandbox tools
- Per-run `SandboxRunConfig` — **client** / live **session** / **session_state** resume / **snapshot** / optional `cwd`

Split of ownership (core design):
- **Outer runtime** owns approvals, tracing, handoffs, resume of the agent run.
- **Sandbox session** owns commands, file changes, and isolation.

Lifecycle: SDK-owned (`client=...`, runner creates/stops) vs developer-owned (`session=...`, you keep the sandbox open across runs). Prefer UnixLocal for macOS/Linux local; Docker/hosted for isolation/parity.

### 4) Guardrails (workflow boundaries matter)
- **Input** guardrails: first agent in the chain only; parallel (default) vs blocking (`run_in_parallel=False` before the expensive model/tools start).
- **Output** guardrails: agent that produces final output only.
- **Tool** guardrails: every guarded function-tool / local MCP tool call; not handoffs; hosted tools have different pipelines.

Tripwire → typed exception; use tool guardrails when managers/handoffs would otherwise skip agent-level I/O checks.

### 5) MCP as tool surface, not a security model
Transports: HostedMCPTool (Responses-side), Streamable HTTP, SSE (legacy), stdio. Filter tools, cache `list_tools`, attach approvals / tool guardrails on local servers. Docs: connect only to trusted servers; least privilege; approvals for sensitive ops. Connectivity ≠ authorization policy.

### 6) Realtime agents
`RealtimeAgent` + `RealtimeRunner` → server-side WebSocket sessions (`gpt-realtime-2.1` recommended in quickstart). Tools, handoffs, output guardrails, approvals still work; structured outputs are not supported; voice cannot change after spoken audio has been produced. Python SDK does not provide browser WebRTC transport.

## How it differs from common approaches
| Approach | Strength | Gap vs Agents SDK (this skill’s focus) |
|----------|----------|----------------------------------------|
| Raw Responses/Chat loop | Full control | You re-implement turns, tools, sessions, tracing |
| **OpenAI Agents SDK** | Fast agent loop; handoffs; guardrails; Sandbox + MCP + Realtime | Not durable-by-construction like a checkpointed graph |
| **LangGraph** | Explicit graph, typed state, checkpointers, HITL/time-travel | Higher ceremony; overkill for simple specialist handoffs |
| **Microsoft Agent Framework** | Azure/.NET enterprise path; AutoGen/SK lineage | Wrong default if you are OpenAI-Python-first and not on that stack |
| Hosted shell tool only | Occasional shell without workspace productization | No Manifest/resume/snapshot lifecycle; use Sandbox when the **workspace boundary is the feature** |
| Dump all MCP tools into context | Easy demo | Context blowup + unsafe surface; filter + menus/approvals needed |

## Limits & failure modes
- Sandbox is **beta** — expect API/default drift.
- Input guardrails on agent A do **not** re-run after handoff to agent B; tool guardrails fill that gap for function/MCP tools.
- Handoff `is_enabled` cannot authorize parsed handoff args; authorize in `on_handoff` and raise on failure.
- Sessions ≠ crash-durable workflow engine; design resume explicitly (RunState / session_state / snapshots) or add external durability.
- MCP expands blast radius; HostedMCPTool runs off your process — approvals and trust boundaries still required.
- Realtime: no structured outputs; interrupt/playback handling is on you when guardrails trip mid-audio.
- `clone()` is shallow — shared mutable lists (`tools`, `handoffs`, …) unless you pass new lists.

## Playbook (do this)

### Personal / solo agent
- [ ] Start with plain `Agent` + one function tool; only add Sandbox when you need a real workspace.
- [ ] For coding/docs tasks: `SandboxAgent` + `Manifest` (`LocalDir` or `GitRepo`) + `UnixLocalSandboxClient` (or Docker on Windows).
- [ ] Put long task specs in workspace files (`repo/task.md`), keep `instructions` short (role, workflow, success criteria).
- [ ] Side-effecting tools: approval or confirm; never treat MCP “connected” as “allowed.”

### Product
- [ ] Choose topology explicitly: handoff (ownership transfer) vs as_tool (nested specialist) vs hybrid (intake → sandbox reviewer).
- [ ] Log: active agent, tool name/args, guardrail tripwires, sandbox session id / snapshot id.
- [ ] Blocking input guardrails for costly or side-effecting entry paths; tool guardrails on send/delete/pay/write.
- [ ] Document sandbox client choice (local vs Docker vs hosted) as an ops decision, not only a DX preference.

### Engineering / MCP
- [ ] Prefer Streamable HTTP or stdio for new MCP; keep SSE only for legacy.
- [ ] `tool_filter` / allowlists; `require_approval` for destructive tools; optional server-wide tool guardrails.
- [ ] Combine Sandbox capabilities with `mcp_servers` when workspace + external policy tools coexist (see SDK `sandbox_agent_with_tools` example pattern).
- [ ] For as_tool sandbox specialists: pass `run_config=RunConfig(sandbox=SandboxRunConfig(...))` per tool when isolation is required; share one session only when read-only explorer semantics are intentional.
- [ ] Regression: handoff does not drop user objective; resume from snapshot still honors path grants from **trusted** manifest (never from untrusted serialized host paths alone).

## Acceptance tests (prove it works)
1. **Sandbox inspect** — Given a Manifest with a known README, When `SandboxAgent` is asked to summarize it via UnixLocal/Docker client, Then the answer cites file content that only exists in the workspace (not hallucinated from name alone).
2. **Handoff ownership** — Given triage → sandbox reviewer handoff, When the user asks a document-heavy question, Then the reviewer becomes the active agent on the **same** run (not a silent nested tool) and cannot “answer without inspecting” if instructions forbid it.
3. **Guardrail tripwire** — Given a blocking input guardrail that rejects off-policy prompts, When such a prompt is sent, Then the main agent/tools never run (`run_in_parallel=False`).
4. **MCP filter** — Given an MCP server exposing read + delete tools, When only read is allowlisted, Then delete is not visible to the model and cannot be invoked.
5. **as_tool isolation** — Given two sandbox tool-agents with separate Docker `RunConfig`s, When both run, Then mutations in one workspace do not appear in the other.

## Suggested X / content angle (optional)
- Hook: Agent 不是 prompt，是 instructions + tools + guardrails + handoffs；需要改真实文件时，再加 Sandbox 这条执行边界。
- Depth: handoff vs as_tool 的控制权差异；Sandbox Manifest/resume；MCP 连通 ≠ 授权；何时改用 LangGraph / Microsoft Agent Framework。
- Asset: 指向本 skill 文件夹的 checklist，而不是只贴 README 徽章。

## Citations
- Repo README core concepts (Agents, Sandbox agents, Realtime, handoffs, tools/MCP, guardrails, HITL, sessions, tracing): https://github.com/openai/openai-agents-python (accessed 2026-09-16).
- Sandbox concepts (beta, Manifest, SandboxRunConfig, capabilities, handoff vs as_tool turn accounting): https://openai.github.io/openai-agents-python/sandbox/guide/
- Agents, guardrails, handoffs, MCP guides on the same docs site (accessed 2026-09-16).
- Star count ~29.5k from the GitHub repository page at authoring time — re-check before citing precisely.
