---
name: state-path-tool-menus
description: >-
  Use this when an agent faces a large tool/MCP library and relevance retrieval
  keeps surfacing the final action without prerequisite producers — design the
  tool menu as an execution prior (state path, producer-before-consumer).
---

# State-Path Tool Menus: The Menu Is an Execution Prior

## Source
- Type: paper (+ code)
- Title: The Menu Is an Execution Prior: State-Path Tool Menus for Online Agents
- Link: https://arxiv.org/abs/2609.09395
- PDF: https://arxiv.org/pdf/2609.09395
- Authors: Bo Yan, Weikai Lin, Song Wang
- Date: submitted 2026-09-08 (arXiv 2609.09395v1)
- Venue note: arXiv comments claim **Accepted to EMNLP 2026 Main** (verify on the abs page / proceedings if you cite venue formally)
- Code: https://github.com/Met2348/State-Path (stars: 未知 / newly public at authoring — do not invent a count)
- One-line gist: Before the agent runs, show a short **ordered** tool menu that covers a complete **state path** (executable entry → missing-input producers → final action) with producers listed before consumers — relevance ranking alone is not enough.

## When to use / when NOT to use

**Use when:**
- The tool/MCP catalog is large (hundreds–thousands) and you already retrieve a short candidate list before each run.
- Multi-step tasks need **prerequisite tools** whose names do not match the user request as strongly as the final action (e.g. lookup order / create receipt / get email **before** send receipt).
- Research agents, ToolBench-like API agents, or MCP gateways where the **menu** (what the model can see) is fixed for the call budget.
- You can keep the online agent loop unchanged and only improve **pre-execution** tool exposure and order.

**Do NOT use when:**
- The agent has ≤ ~10 tools and all are always in context — menu construction is overhead.
- Tools are pure single-step and have no producer/consumer dependencies.
- You need **observation-conditioned** tool set updates every step as the primary mechanism (paper fixes one menu before execution; online adaptive planners are complementary, not replaced).
- Schemas lack usable input/output fields and you have no trajectory history — constructor quality degrades (paper limitation).

## Problem (plain language)
Agents act through tools, but practical libraries are huge (ToolBench-scale: abstract/intro note **>16,000 APIs**). Systems therefore show the model a short **tool menu**: the ordered subset it may call.

Most constructors rank by **request relevance**. On multi-step tasks that fails structurally: the final action (e.g. SendEmailReceipt) looks most relevant, so it rises to the top **before** inputs like `email` or `receipt_id` exist. The agent then fails immediately or wastes the call budget, even though the right producers were somewhere lower in the list — or omitted entirely.

The conflict is **individual relevance vs route completeness**. The menu is not just search results; it is an **execution prior** over possible routes from the current observable state to the desired outcome.

## Core mechanism (how it works)

### 1) Define the tool menu
- Short, **ordered** subset of interfaces shown before execution.
- Agent may call **only** tools on this menu.
- Selection and order are fixed for the execution (paper setting) while the online loop and call budget stay the same.

### 2) Define the state path
A pre-execution route through tool-induced states:
1. Start from **visible** request fields (observable state).
2. Cross tools that **produce missing inputs** (bridges / producers).
3. End at the **requested outcome** tool (consumer / target).

Relevance scores cannot express whether selected tools form a complete, runnable route. The state path makes that route the construction objective.

### 3) State-Path Tool Menu (encoder → retriever → reranker)
- **Encoder**: relation-aware tool representations from request state, tool schemas, and training paths — what can run now, how outputs satisfy later inputs, which orders recur.
- **Retriever**: cover complementary path roles — an **executable entry**, **missing-input producers**, and the **final action** (paper uses a **32-tool** menu budget in the main ToolBench setting).
- **Reranker**: place **producers before consumers** so the displayed order forms an executable prefix.

The online agent is **unchanged**; only the menu improves.

### 4) Verified ToolBench signal (from abstract / PDF)
On ToolBench, the State-Path menu raises **online success from 0.737 to 0.898** (paper: +49 successful executions in the matched 32-tool-budget setting shown in Fig. 1 / conclusion), outperforms retrieval / reranking / generation / routing baselines **without changing the agent**, covers more complete chains with **32 tools** than the official list with **128**, and the success gain persists across executor families with different model capacities. Treat as paper-reported — re-measure on your stack.

## How it differs from common approaches
| Approach | Strength | Gap vs State-Path |
|----------|----------|-------------------|
| Always dump full catalog | Simple | Context limits; model cannot see a route in thousands of tools |
| Relevance retrieval / ToolRet-style | Good query–tool text match | Surfaces destination; omits/delays producers |
| Unordered coverage (e.g. COLT-like complete sets) | Better recall of needed tools | Order still wrong → non-executable top of menu |
| Generation / routing of tool ids | Removes external retriever | Still not optimizing state-path completeness + producer-before-consumer display |
| Online adaptive planners (update tools mid-run) | Reacts to observations | Complements State-Path; does not replace a good **initial** menu |
| **State-Path menu** | Route-complete, ordered execution prior; agent unchanged | Needs schemas/paths; one menu fixed pre-execution; build cost |

## Limits & failure modes
- **One menu before execution** — no observation-conditioned menu updates in the paper method (limitation §6).
- Depends on **informative input/output fields** and recurring path relations; sparse/corrupt histories degrade gradually.
- Cross-library transfer helps entry coverage but full-chain coverage still benefits from local trajectories (paper transfer discussion).
- Better menus can also make **harmful** tool sequences easier to see/execute — pair with permission checks, sandboxing, and human review (ethics statement).
- Numbers (0.737 → 0.898, 32 vs 128) are **not portable** without your own eval.

## Playbook (do this)

### Personal / solo agent / research agent
- [ ] Inventory tools as a dependency graph: for each tool, list required inputs and produced outputs (from schema fields, not marketing descriptions alone).
- [ ] For each request, sketch the state path on paper: visible fields → missing producers → final action.
- [ ] Build the menu so the **first callable** tools are ones whose inputs are already satisfied (executable entry), then bridges, then target.
- [ ] Cap menu size (e.g. 16–32); spend budget on **route coverage**, not on near-duplicate relevance hits.

### Product
- [ ] Treat “tool retrieval” as **menu construction**: log `menu_ids`, `menu_order`, `path_roles` (entry/bridge/target), and whether the first call was schema-feasible.
- [ ] Metric beyond recall@k: **executable entry rate**, **complete-chain coverage@k**, **producer-before-consumer violations**.
- [ ] A/B: relevance-only menu vs state-path-ordered menu with the **same** agent and call budget.
- [ ] Safety: menu construction must not auto-include high-impact tools without policy; combine with SARA-style authorization for side effects.

### Engineering / MCP
- [ ] MCP `list_tools` → your gateway should **filter + order**, not forward the entire server catalog.
- [ ] Prefer schemas with explicit input/output fields; paper Table 9: names+I/O fields beat names-only for dependency edges / replay (structured fields ≈ full descriptions on their measures).
- [ ] Static allowlists are coarse; add a **per-request menu** layer for large MCP meshes (calendar + docs + code + browser…).
- [ ] Optional: train/adapt retriever+reranker on your traces; until then, heuristic state-path (schema BFS from visible slots + topological producer-before-consumer sort) is already an upgrade over pure embedding rank.
- [ ] Keep online planners free to revise after observations — State-Path supplies the **initial** prior.

## Acceptance tests (prove it works)
1. **Producer-before-consumer** — Given a receipt-email style task with visible `order_id` only, When the menu is built, Then lookup/create/email-producer tools appear **before** the send/receipt consumer, and the consumer is still present within the budget.
2. **Relevance trap** — Given a relevance baseline that ranks the final action #1 with missing inputs, When State-Path (or your heuristic) builds the menu, Then the first tool is schema-feasible from observable state (executable entry).
3. **Budget fairness** — Given the same agent and call budget, When only the menu constructor changes, Then task success / complete-chain coverage improves or you document why (missing schemas, etc.).
4. **MCP mesh** — Given three MCP servers totaling >100 tools, When a research question needs “search → fetch → cite”, Then the menu includes all three roles in order, not only the cite tool that lexically matches “write report.”

## Suggested X / content angle (optional)
- Hook: 给 Agent 的不是「最相关的工具」，而是一条可执行的 **state path**——producer 必须排在 consumer 前面。
- Depth: 为何 relevance 会把最终动作顶到第一却缺输入；菜单作为 execution prior；ToolBench 0.737→0.898（需标注论文设定）；MCP 网关怎么落地。
- Asset: 本 skill 的 checklist + 论文/代码链接。

## Citations
Verified from https://arxiv.org/abs/2609.09395 and PDF https://arxiv.org/pdf/2609.09395 (accessed 2026-09-16): tool-menu definition; state-path definition; encoder/retriever/reranker; producer-before-consumer; ToolBench online success **0.737 → 0.898**; more complete chains with **32** tools than official **128**; gains across executor families; code URL https://github.com/Met2348/State-Path; abs comments **Accepted to EMNLP 2026 Main**; limitations on fixed pre-execution menu and schema/path dependence. Re-verify tables before citing finer ablations.
