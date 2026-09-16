# AI Agent Skills

Reusable Agent Skills distilled from papers and open-source repos — focused on **AI Agents 实操** (workflows, pitfalls, authorization, tooling).

Each skill is a folder with a Cursor-compatible `SKILL.md` (YAML frontmatter + recipe). Install by copying into your agent skills directory, or by pointing your agent at this repo.

## Skills

| Skill | Source | When to use |
|-------|--------|-------------|
| [sara-tool-authorization](./skills/sara-tool-authorization/) | [arXiv:2608.27146](https://arxiv.org/abs/2608.27146) | Designing or auditing tool-using agents so tool *outputs* cannot silently become *authorization* |
| [openai-agents-sdk-sandbox](./skills/openai-agents-sdk-sandbox/) | [openai/openai-agents-python](https://github.com/openai/openai-agents-python) | Building with OpenAI Agents SDK: SandboxAgent workspaces, MCP, guardrails, handoffs, Realtime — and when to prefer LangGraph / Microsoft Agent Framework |
| [state-path-tool-menus](./skills/state-path-tool-menus/) | [arXiv:2609.09395](https://arxiv.org/abs/2609.09395) | Large tool/MCP libraries: build the tool *menu* as an execution prior (state path, producer-before-consumer), not relevance-only retrieval |

## Add a skill (pipeline)

1. Read the paper or repo thoroughly (problem, method, results, limits).
2. Copy `templates/SKILL.template.md` → `skills/<kebab-name>/SKILL.md`.
3. Fill: when to use, mechanism, comparisons, limits, checklists, acceptance tests.
4. Link the original source; never invent citations or numbers.
5. Open a PR.

## Install (Cursor / Grok Bot style)

Place `skills/<name>/` where your agent loads skills, or paste `SKILL.md` into your skills library. The `description` frontmatter is what agents use to decide when to apply it.

## License

MIT — skill text is yours to reuse; paper/repo rights stay with original authors. Always cite the source.
